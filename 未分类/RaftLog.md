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
Message StateMachineLogEntryProto {
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

还有一点需要注意的是，SegmentFileInfo也是value-based class。这里是这么理解newEndIndex的，因为SegmentFileInfo对象是value-based，其实例变量的状态在构造后不可改变。当对一个LogSegment进行truncate需要返回一个新的segmentFileInfo对象，这个对象记录了其endIndex，即truncate前的index，以及truncate之后的index，即newEndIndex，以及turncate的position/offset，即targetLength。可见SegmentFileInfo属于是瞬时对象，不断地创建和销毁，而且是一次性的。

```java
static final class SegmentFileInfo {
    static SegmentFileInfo newClosedSegmentFileInfo(LogSegment ls) {
        Preconditions.assertTrue(!ls.isOpen());
        return new SegmentFileInfo(ls.getStartIndex(), ls.getEndIndex(), ls.isOpen(), 0, 0);
    }
    
    private final long startIndex;
    private final long endIndex; //original end index
    private final boolean isOpen;
    private final long targetLength; //position for truncation
    private final long newEndIndex; //new endIndex after the truncation
    
    private SegmentFileInfo(long start, long end, boolean isOpen, long targetLength, long newEndIndex) {
        this.startIndex = start;
        this.endIndex = end;
        this.isOpen = isOpen;
        this.targetLength = targetLength;
        this.newEndIndex = newEndIndex;
    }
    
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

### 3. LogSegmentList

持有一个LogSegment的list，每个LogSegment对象是一个SegmentedRaftLog文件在内存中的镜像。这里理一下，每个LogSegment持有一裤兜子LogRecord，一个LogSegmentList持有一裤兜子LogSegment

实例变量比较简单

* List\<LogSegment> segments = new ArrayList<>()
* AutoCloseableReadWriteLock lock
* long sizeInBytes

构造器

```java
LogSegmentList(Object name) {
    this.name = name;
    this.lock = new AutoCloseableReadWriteLock(name);
    this.sizeInBytes = 0;
}
```

其isEmpty() size()方法针对的都是segments这个list对象的操作

这里第一个 关注点是getTermIndex(long, long, LogSegment)方法

```java
LogEntryHeader[] getTermIndex(long startIndex, long realEnd, LogSegment openSegment) {
    //构造LogEntryHeader空数组
    final LogEntryHeader[] entries = new LogEntryHeaderj[Math.toIntExact(realEnd - startIndex)];
    final int searchIndex;
    long index = startIndex;
    try(AutoClsoeableLock readLock = readLock()) {
        searchIndex = Collection.binarySearcyh(segmetns, startIndex);
        if(searchIndex > 0) { //segments这个list包含了startIndex
            for(int i = searchIndex; i < segments.size() && index < realEnd; i ++) {
                //找到起始的LogSegment
                final LogSegment s = segments.get(i);
                final int numerFromSegment = Math.toIntExact(Math.min(realEnd - index, s.getEndIndex() - index + 1));
                getFromSegment(s, index, entries, Math.toIntExact(index - startIndex), numberFromSegment);
                index += numberFromSegment;
            }
        }
    }
    
    if(searchIndex < 0) {
        //binarySearch并没有搜到，换言之segments中并没有包含startIndex的LogSegment
        getFromSegment(openSegment, startIndex, entries, 0, entries.length);
    } else { //搜到了，但是可能没搜完
        getFromSegment(openSegment, startIndex, entries, Math.toIntExact(index, startIndex), Math.toIntExact(realEnd - index));
    }
    
    return entries;
}
```

add(LogSegment)方法只是单纯的向segments中增加一个LogSegment对象，更新SegmenetedRaftLogCache的sizeInBytes信息

```java
boolean add(LogSegment logSegment) {
    try(AutoCloseable writeLock = writeLock()) {
        sizeInBytes += logSegment.getTotalFileSize();
        return segments.add(logSegment);
    }
}
```

### 4. 实例变量

持有一个正在写的LogSegment和一裤兜子已经写完的LogSegment

```java
private final String name;
private volatile LogSegment openSegment;
private final LogSegmentList closedSegments;
private final RaftStorage storage;
private final int maxCachedSegments;
private final CacheInvalidationPolicy evictionPolicy = new CacheInvalidationPolicyDefault();
private final long maxSegmentCacheSize;
```



### 5. getFromSegment(LogSegment, long, LogEntryHeader[], int, int)方法

这里是从LogSegment中提取多个LogRecord，从startIndex开始后的size个，将其放入到LogEntryHeader中开始的位置上

首先，比对LogSegment中最后一个LogEntry的index和想要提取的LogRecord的数量，取LogSegment中剩余LogRecord的数量和size的最小值作为循环结束条件。遍历LogSegment中的LogRecord，提取其EntryHeader信息放入到LogEntryHeader数组的对应位置上。

这里不会出现数组越界的情况，因为对endIndex做了边界的考虑。

```java
private static void getFromSegment(LogSegment segment, long startIndex, LogEntryHeader[] entries, int offset, int size) {
    long endIndex = segment.getEndIndex();
    endIndex = Math.min(endIndex, startIndex + size - 1);
    int index = offset;
    for (long i = startIndex; i < endIndex; i ++) {
        entries[index ++] = Optinal.ofNullable(segment.getLogRecord(i)).map(LogRecord::getLogEntryHeader).orElse(null);
    }
}
```



## 7. SegmentedRaftLog

入口方法在appendEntryImpl(LogEntryProto)中

```java
  @Override
  public List<CompletableFuture<Long>> appendImpl(List<LogEntryProto> entries) {
    checkLogState();
    if (entries == null || entries.isEmpty()) {
      return Collections.emptyList();
    }
    try(AutoCloseableLock writeLock = writeLock()) {
      final TruncateIndices ti = cache.computeTruncateIndices(server::notifyTruncatedLogEntry, entries);
      final long truncateIndex = ti.getTruncateIndex();
      final int index = ti.getArrayIndex();
      LOG.debug("truncateIndex={}, arrayIndex={}", truncateIndex, index);

      final List<CompletableFuture<Long>> futures;
      if (truncateIndex != -1) {
        futures = new ArrayList<>(entries.size() - index + 1);
        futures.add(truncate(truncateIndex));
      } else {
        futures = new ArrayList<>(entries.size() - index);
      }
      for (int i = index; i < entries.size(); i++) {
        futures.add(appendEntry(entries.get(i))); //这里是入口
      }
      return futures;
    }
  }
```

appednEntry(LogEntryProto)方法调用appendEntryImpl(LogEntryProto)方法

```java
  protected CompletableFuture<Long> appendEntryImpl(LogEntryProto entry) {
    final Timer.Context context = getRaftLogMetrics().getRaftLogAppendEntryTimer().time();
    checkLogState();
    if (LOG.isTraceEnabled()) {
      LOG.trace("{}: appendEntry {}", getName(), LogProtoUtils.toLogEntryString(entry));
    }
    try(AutoCloseableLock writeLock = writeLock()) {
      validateLogEntry(entry);
      final LogSegment currentOpenSegment = cache.getOpenSegment();
      if (currentOpenSegment == null) {
        cache.addOpenSegment(entry.getIndex());
        fileLogWorker.startLogSegment(entry.getIndex());
      } else if (isSegmentFull(currentOpenSegment, entry)) {
        cache.rollOpenSegment(true);
        fileLogWorker.rollLogSegment(currentOpenSegment);
      } else if (currentOpenSegment.numOfEntries() > 0 &&
          currentOpenSegment.getLastTermIndex().getTerm() != entry.getTerm()) {
        // the term changes
        final long currentTerm = currentOpenSegment.getLastTermIndex().getTerm();
        Preconditions.assertTrue(currentTerm < entry.getTerm(),
            "open segment's term %s is larger than the new entry's term %s",
            currentTerm, entry.getTerm());
        cache.rollOpenSegment(true);
        fileLogWorker.rollLogSegment(currentOpenSegment);
      }

      //TODO(runzhiwang): If there is performance problem, start a daemon thread to checkAndEvictCache
      checkAndEvictCache();

      // If the entry has state machine data, then the entry should be inserted
      // to statemachine first and then to the cache. Not following the order
      // will leave a spurious entry in the cache.
      CompletableFuture<Long> writeFuture =
          fileLogWorker.writeLogEntry(entry).getFuture();
      if (stateMachineCachingEnabled) {
        // The stateMachineData will be cached inside the StateMachine itself.
        cache.appendEntry(LogProtoUtils.removeStateMachineData(entry),
            LogSegment.Op.WRITE_CACHE_WITH_STATE_MACHINE_CACHE);
      } else {
        cache.appendEntry(entry, LogSegment.Op.WRITE_CACHE_WITHOUT_STATE_MACHINE_CACHE);
      }
      return writeFuture;
    } catch (Exception e) {
      LOG.error("{}: Failed to append {}", getName(), LogProtoUtils.toLogEntryString(entry), e);
      throw e;
    } finally {
      context.stop();
    }
  }
```

这几涉及到了之前要用到的SegmentedRaftLogCache和SegmentedRaftLogWorker的相关方法