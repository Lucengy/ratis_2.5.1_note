## 1. RaftLogSequentialOps

有关Runner类，Ratis-406的说法是

```
runSequentially does nothing but only an assertion(asserting only at most one thread at a time).
```

接口方法

```java
  //Used by leader
  long append(long term, TransactionContext transaction) throws StateMachineException;
  //Used by leader
  long append(long term, RaftConfiguration configuration);
  //Used by leader
  long appendMetadata(long term, long commitIndex);
  //Used by the leader and the followers.
  CompletableFuture<Long> appendEntry(LogEntryProto entry);
  //Used by the followers.
  List<CompletableFuture<Long>> append(List<LogEntryProto> entries);
  //Used by the leader and the followers.
  CompletableFuture<Long> truncate(long index);
```

## 2. RaftLog

继承自RaftLogSequentialOps和Closeable接口

以下来自Ratis-122

![1736779628165](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1736779628165.png)

![1736779660412](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1736779660412.png)

在Raft.proto文件中，LogEntry的类型目前有三种，如下所示

```protobuf
message LogEntryProto {
  uint64 term = 1;
  uint64 index = 2;

  oneof LogEntryBody {
    StateMachineLogEntryProto stateMachineLogEntry = 3;
    RaftConfigurationProto configurationEntry = 4;
    MetadataProto metadataEntry = 5;
  }
}
```

针对写请求的logEntry，其类型应该是StateMachineLogEntryProto，这里包含RaftLog的内容，同时，做了StateMachine data和RaftLog data的分离，那么StateMachineLogEntryProto的定义为

```protobuf
essage StateMachineLogEntryProto {
  // TODO: This is not super efficient if the SM itself uses PB to serialize its own data for a
  /** RaftLog entry data */
  bytes logData = 1;
  /**
   * StateMachine entry.
   * StateMachine implementation may use this field to separate StateMachine specific data from the RaftLog data.
   */
  StateMachineEntryProto stateMachineEntry = 2;

  enum Type {
    WRITE = 0;
    DATASTREAM = 1;
  }

  Type type = 13;
  // clientId and callId are used to rebuild the retry cache.
  bytes clientId = 14;
  uint64 callId = 15;
}
```

logData是必有的，这里用来表示的是RaftLog的数据，stateMachineData是可选的，其定义为

```protobuf
message StateMachineEntryProto {
   /**
    * StateMachine specific data which is not written to log.
    * Unlike logEntryData, stateMachineData is managed and stored by the StateMachine but not the RaftLog.
    */
  bytes stateMachineData = 1;
   /**
    * When stateMachineData is missing, it is the size of the serialized LogEntryProto along with stateMachineData.
    * When stateMachineData is not missing, it must be set to zero.
    */
  uint32 logEntryProtoSerializedSize = 2;
}
```

看完这里，再来看RaftLog总的内部类EntryWithData

```java
interface EntryWithData {
    int getSerializedSize();
    LogEntryProto getEntry(TimeDuration timeout) throws RaftLogIOException, TimeoutExcepiton;
}
```

接下来是RaftLog的接口

```java
public interface RaftLog extends RaftLogSequentialOps, Closeable {
    default boolean contains(TermIndex ti) {
        return ti.equals(getTermIndex(ti.getIndex()));
    }
    TermIndex getTermIndex(long index);
    LogEntryProto get(long index) throws RaftLogIOException;
    EntryWithData getEntryWithData(long index) throws RaftLogIOException;
    long getStartIndex();
    
    default long getNextIndex() {
        final TermIndex last = getLastEntryTermIndex();
        if(last == null){
            //if the log is empty, the last committed index should be consistent with
            //the last index included in the latest snapshot.
            return getLastCommittedIndex() + 1;
        }
        
        return last.getIndex() + 1;
    }
    
    long getLastCommittedIndex(); //return the index of the last entry that has been committed.
    
    long getSnapshotIndex();
    
    // @return the index of the last entry that has been flushed to the local storage.
    long getFlushIndex();
    
    TermIndex getLastEntryTermIndex();
    
    boolean updateCommitIndex(long majorityIndex, long curerntTerm, boolean isLeader);
    
    void updateSnapshotIndex(long newSnapshotIndex);
    
    void open(long lastIndexInSnapshot, Consumer<LogEntryProto> consumer) throws IOException;
    
    CompletableFuture<Long> purge(long suggestedIndex);
    
    void persistMetadata(RaftStorageMetadata metadata) throws IOException;
    
    RaftStorageMetadata loadMetadata() throws IOException;
    
    CompletableFuture<Long> onSnapshotInstalled(long lastSnapshotIndex);
}
```

## 3. RaftLogBase

首先是EntryWithData的实现类

```java
class EntryWithDataImpl implements EntryWithData {
    private final LogEntryProto logEntry;
    private final CompletableFuture<ByteString> future;
    
    EntryWithDataImpl(LogEntryProto logEntry, CompletableFuture<ByteString> future) {
        this.logEntry = logEntry;
        //实例::实例方法。实参全部传递给形参
        this.future = future == null? null: future.thenApply(this::checkStateMachineData);
    }
    
    private ByteString checkStateMachineData(BytString data) {
        if(data == null) {
            throw new IllegalStateException("state machine data is null for log entry");
        }
        return data;
    }
    
    @Override
    public int getSerializedSize() {
        return LogProtoUtils.getSerializedSize(logEntry);
    }
    
    @Override
    public LogEntryProto getEntry(TimeDuration timeout) throws RaftLogIOException, TimeoutException {
        LogEntryProto entryProto;
        if(future == null) {
            return logEntry;
        }
        
        try {
            entryProto = future.thenApply(data -> LogProtoUtils.addStateMachineData(data, logEntry))
                .get(timeout.getDuration(), timeout.getUnit());
        } catch(TimeoutException t) {
            if(timeout.compareTo(stateMachineDataReadTimeout) > 0) {
                getRaftLogMetrics().onStateMachineDataReadTimeout();
            }
            throw t;
        } catch(Exception e) {
            throw new RaftLogIOException();
        }
        
        return entryProto;
    }
}
```

这里用到的LogProtoUtils中的几个方法，记录如下(Ratis-353)

```java
private static Optional<StateMachineEntryProto> getStateMachineEntry(LogEntryProto entry) {
    return Optional.of(entry)
        .filter(LogEntryProto::hasStateMachineLogEntry)
        .map(LogEntryProto::getStateMachineLogEntry)
        .filter(StateMachineLogEntryProto::hasStateMachineEntry)
        .map(StateMachineLogEntryProto::getSerializedSize);
}

public static int getSerializedSize(LogEntryProto entry) {
    return getStateMachineEntry(entry)
        .filter(stateMachineEntry -> stateMachineEntry.getStateMachineData().isEmpty())
        .map(StateMachineEntryProto::getLogEntryProtoSerializedSize)
        .orElseGet(entry::getSerializedSize);
}

static LogEntryProto addStateMachineData(ByteString stateMachineData, LogEntryProto entry) {
    return replaceStateMachineEntry(entry, StateMachineEntryProto.newBuilder().setStateMachineData(stateMachineData));
}

static LogEntryProto replaceStateMachineEntry(LogEntryProto proto, StateMachineEntryProto.Builder newEntry) {
    return LogEntryproto.newBuilder(proto)
        .setStateMachineLogEntry(StateMachineLogEntryProto.newBuilder(proto.getStateMachineLogEntry()).setStateMachineEntry(newEntry)).build();
}
```



抽象类

```java
public abstract class RaftLogBase implements RaftLog {
    private final String name;
    private final RaftLogIndex commitIndex;
    private final RaftLogIndex snapshotIndex;
    private final RaftLogIndex purgeIndex;
    private final int purgeGap;
    private final RaftGroupMemberId memberId;
    private final int maxBufferSize;
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock(true);
    private final Runner runner = new Runner(this::getName);
    private final OpenCloseState state;
    private final LongSupplier getSnapshotIndexFromStateMachine;
    private final TimeDuration stateMachineDataReadTimeout;
    private final long purgePreservation;
    private volatile LogEntryProto lastMetadataEntry = null;
    
    @Override
    public boolean updateCommitIndex(long majority, long currentTerm, boolean isLeader) {
        try(AutoCloseableLock writeLock = writeLock()) {
            final long oldCommittedIndex = getLastCommittedIndex();
            final long newCommitIndex = Math.min(majorityIndex, getFlushIndex());
            if(oldCommittedIndex < newCommitIndex) {
                if(!isLeader) {
                    commitIndex.updateIncreasingly(newCommitIndex, traceIndexChange);
                    return true;
                }
                
                final TermIndex entry = getTermIndex(newCommitIndex);
                if(entry != null && entry.getTerm() == currentTerm) {
                    commitIndex.updateIncreasingly(newCommitIndex, traceIndexChange);
                    return true;
                }
            }
        }
        
        return false;
    }
    
    @Override
    public void updateSnapshotIndex(long newSnapshotIndex) {
        try(AutoCloseableLock writeLock = writeLock()) {
            final long oldSnapshotIndex = getSnapshotIndex();
            if(oldSnapshotIndex < newSnapshotIndex) {
                snapshotIndex.updateIncreasingly(newSnapshotIndex, infoIndexChange);
            }
            
            final long oldCommitIndex = getLastCommittedIndex();
            if(oldCommitIndex < newSnapshotIndex) {
                commitIndex.updateIncreasingly(newSnapshotIndex, traceIndexChange);
            }
        }
    }
    
    @Override
    public final long append(long term, TransactionContext transaction) throws StateMachineException {
        return runner.runSequentially(() -> appendImpl(term, transaction));
    }
    
    private long appendImpl(long term, TransactionContext operation) throws StateMachineException {
        checkLogState();
        
        try(AutoCloseableLock writeLock = writeLock()) {
            final long nextIndex = getNextIndex();
            try {
                operation = operation.preAppendTransaction();
            } catch (StateMachineException e) {
                throw e;
            } catch (IOException e) {
                throw new StateMachineException(memberId, e);
            }
            //这里并没有剔除掉StateMachineData
            final LogEntryProto e = operation.initLogEntry(term, nextindex);
            int entrySize = e.getSerializedSize();
            
            appendEntry(e);
            return nextIndex;
        }
    }
    
    public final void open(long lastIndexInSnapshot, Consumer<LogEntryProto> consumer) throws IOException {
        openImpl(lastIndexInSnapshot, e -> {
            if(e.hasMetadataEntry()) {
                lastMetadataEntry = e;
            } else if(consumer != null) {
                consumer.accept(e);
            }
        });
        
        Optional.ofNullable(lastMetadataEntry).ifPresent(
          e -> commitIndex.updateToMax(e.getMetadataEntry().getCommitIndex(), infoIndexChange));
        state.open();
        
        final long startIndex = getStartIndex();
        if(startIndex > LEAST_VALID_LOG_INDEX) {
            purgeIndex.updateIncreasingly(startIndex - 1, infoIndexChange);
        }
    }
    
    protected void openImpl(long lastIndexInSnapshot, Consumer<LogEntryProto> consumer) throws IOException {}
    
    protected void validateLogEntry(LogEntryProto entry) {
        if(entry.hasMetadataEntry())
            return;
        
        long latestSnapshotIndex = getSnapshotIndex();
        TermIndex lastTermIndex = getLastEntryTermIndex();
        
        if(lastTermIndex != null) {
            long lastIndex = lastTermIndex.getIndex() > latestSnapshotIndex ?
                lastTermIndex.getIndex() : latestSnapshotIndex;
        } else {
            Preconditions.assertTrue(entry.getIndex == latestSnapshotIndex + 1);
        }
    }
    
    @Override
    public final CompletableFuture<Long>
}
```

有关updateCommitIndex(long, long, boolean)部分请看RATIS-748，以下为修改前的代码

![1736864842790](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1736864842790.png)

以下为问题的描述

![1736864892577](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1736864892577.png)

问题点在于updateCommitIndex(long, long, boolean)方法会被leader和follower均使用，在follower上时，如果出现上图的情况，那么

```java
final TermIndex entry = getTermIndex(majorityIndex)
```

这里会返回null，导致follower本应将committedIndex更新到1371却一直停留都在269

## 4. LogSegment

这是LogSegment的buffer/cache，所有修改先写到LogSegment中，然后按序写到对应的文件中。



## 5. RaftLogSegmentStartEnd

```
This is a value-based class
```

![1736898935507](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1736898935507.png)

首先看以下构造器，以及静态的valueOf()相关的method。这里的getOpenLogFileName(long)和getClosedLogFileName(long, long)并没有做校验，而是将文件名进行拼装，直接返回字符串

```java
private final long startIndex;
private final long endIndex;

private LogSegmentStartEnd(long startIndex, long endIndex) {
    Preconditions.assertTrue(startIndex >= RaftLog.LEAST_VALID_LOG_INDEX);
    Preconditions.assertTrue(endIndex == null || endIndex >= startIndex);
    this.startIndex = startIndex;
    this.endIndex = endIndex;
}

static LogSegmentStartEnd valueOf(long startIndex) {
    return new LogSegmentStartEnd(startIndex, null); //可以将null赋值给long类型，amazing
}

static LogSegmentStartEnd valueOf(long startIndex, long endIndex) {
    return new LogSegmentStartEnd(startIndex, endIndex);
}

static LogSegmentStartEnd valudOf(long startIndex, long endIndex, boolean isOpen) {
    return new LogSegmentStartEnd(startIndex, isOpen? null: endIndex);
}

private String getFileName() {
    return isOpen()? getOpenLogSegmentFileName(startIndex): getClosedLogFileName(startIndex, endIndex);
}

private static String getClosedLogFileName(long startIndex, long endIndex) {
    return LOG_FILE_NAME_PREFIX + "_" + startIndex + "-" + endIndex;
}

private static String getOpenLogFileName(long startIndex) {
    return LOG_FILE_NAME_POREFIX + "_" + IN_PROGRESS + "_" + startIndex;
}
```

这里的isOpen方法是通过endIndex是否为null衡量的

```java
public boolean isOpen() {
    return endIndex == null;
}
```

因为LogSegmentStartEnd是不可变的类，那么构造之初，其实例变量的状态就是固定不变的，那么盲猜一手当logSegment从IN_PROCESS到CLOSED状态转变的过程中，实际上是要构造两个LogSegmentStartEnd对象的。接下来就剩下getFile()相关方法了，实际上只是通过实例变量的状态返回对应的file

```java
private String getFileName() {
    return isOpen()? getOpenLogFileName(startIndex): getClosedLogFileName(startIndex, endIndex);
}

File getFile(File dir) {
    return new File(dir, getFileName());
}

File getFile(RaftStorage storage) {
    return getFile(storage.getStorageDir().getCurrentDir());
}
```

## SegmentedRaftLogCache

### 1. 内部类SegmentFileInfo

几乎与SegmentedRaftLog的实例一样，只是多了targetLength和newEndIndex

targetLength用来指示position for truncation

newEndIndex用来指示new end index after the truncation

还有一点需要注意的是，SegmentFileInfo也是value-based class

```java
static final class SegmentFileInfo {
    static SegmentFileInfo newClosedSegmentFileInfo(LogSegment ls) {
        Preconditions.assertTrue(!ls.isOpen());
        return new SegmentFileInfo(ls.getStartIndex(), ls.getEndIndex(), ls.isOpen(), 0, 0);
    }
    
    private final long startIndex;
    private final long endIndex;
    private final boolean isOpen;
    private final long targetLength;
    private final long newEndIndex;
    
    File getFile(RaftStorage storage) {
        return LogSegmentStartEnd.valueOf(startIndex, endIndex, isOpen).getFile(storage);
    }
    
    File getNewFile(RaftStorage storage) {
        return LogSegmentStartEnd.valueOf(startIndex, endIndex, false).getFile(storage);
    }
}
```

### 2. 内部类TruncationSegments

这里需要思考的一点是，假设RaftLog文件写错了，需要truncate，但是可能已经形成了多个closed状态的RaftLog文件，那么我们需要将待truncate的index之后形成的RaftLog文件均删除。总之，一句话，待删除的RaftLog文件可能有0个至多个，但是需要truncate的文件只有0个或1个

```java
static class TruncationSegments {
    private final SegmentFileInfo toTruncate;
    private final SegmentFileInfo[] toDelete;
    
    TruncationSegments(SegmentFileInfo toTruncate, List<SegmentFileInfo> toDelete) {
        this.toDelete = toDelete == null ? null: toDelete.toArray(new SegmentFileInfo[toDelete.size()]);
        this.toTruncate = toTrucnate;
    }
    
    //Ratis-563
    long maxEndIndex() {
        long max = Long.MIN_VALUE;
        
        if(toTruncate != null) {
            max = toTruncate.endIndex;
        }
        for(SegmentFileInfo d : toDelete) {
            max = Math.max(max, d.endIndex);
        }
        
        return max;
    }
}
```

## 7. SegmentedRaftLog

入口方法在appendEntryImpl(LogEntryProto)中