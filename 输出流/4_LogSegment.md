## 1. 前言

这里涉及到LogEntryHeader一个外部类，LogRecord和LogEntryLoader两个内部类

## 2. LogEntryHeader

LogEntryHeader意义上来讲是每一个logEntry的元数据信息，在内存中用来保存其term index和entry类型这三种元数据

接口，提供了默认的实现。主要提供了三个API，分别为

* getTerm()
* getIndex()
* getLogEntryBodyCase

其中EntryBodyCase也包括了三种

```protobuf
  oneof LogEntryBody {
    StateMachineLogEntryProto stateMachineLogEntry = 3;
    RaftConfigurationProto configurationEntry = 4;
    MetadataProto metadataEntry = 5;
  }
```

这里需要注意的时LogEntryHeader定位仪为一个interface

```java
public interface LogEntryHeader extends Comparable<LogEntryHeader> {
    LogEntryHeader[] EMPTY = {}
    
    TermIndex getTermIndex();
    
    default long getTerm() {
        return getTermIndex().getTerm();
    }
    
    default long getIndex() {
        return getTermIndex().getIndex();
    }
    
    LogEntryProtoCase getLogEntryBodyCase();
    
    static LogEntryHeader valueOf(LogEntryProto entry) {
        return valueOf(TermIndex.valueOf(entry), entry.getLogEntryBodyCase());
    }
    
    //通过匿名内部类返回
    static LogEntryHeader valueOf(TermIndex ti, LogEntryBodyCase logEntryBodyCase) {
        return new LogEntryHeader() {
            //简单的重写方法
        }
    }
}
```

## 3. LogRecord

LogRecord意义上讲保存的是每一条LogEntry的元数据信息，在LogEntryHeader的基础上，封装了文件层，即该LogEntry在文件中的偏移量offset，持有两个实例变量，分别是offset和LogEntryHeader

```java
private final long offset;
private final LogEntryHeader logEntryHeader;
```

## 4. LogEntryLoader

当LogEntry层面发生cahe miss时，从文件中加载LogEntry到内存中，以下代码为LogSegment的实例方法

```java
  synchronized LogEntryProto loadCache(LogRecord record) throws RaftLogIOException {
    LogEntryProto entry = entryCache.get(record.getTermIndex());
    if (entry != null) {
      return entry;
    }
    try {
      return cacheLoader.load(record);
    } catch (Exception e) {
      throw new RaftLogIOException(e);
    }
  }
```

当cache miss时，直接调用cacheLoader.loader(LogRecord)方法，那么涉及到list和map的调整也都在cacheLoader.load(LogRecord)方法中

```java
class LogEntryLoader extends CacheLoader<LogRecord, LogEntryProto> {
    @Override
    public LogEntryProto load(LogRecord key) throws IOException {
        //getFile从实例变量的状态获取到对应的SegmentedRaftLogFile
        final File file = getFile();
        final AtomicReference<LogEntryProto> toReturn = new AtomicReference<>();
        //这里是外部类LogSegment的静态方法，所以要传入实例变量的状态，最后一个参数是Consumer<LogEntryProto>
        readSegmentFile(file, startIndex, endIndex, isOpen, getLogCorruptionPolicy(), raftLogMetrics, entry -> {
            final TermIndex ti = TermIndex.valueOf(entry);
            putEntryCache(ti, entry, Op.LOAD_SEGMENT_FILE);
            if(ti.equals(key.getTermIndex())){
                toReturn.set(entry);
            }
        });
        loadingTimes.incrementAndGet();
        return Objects.requireNonNull(toReturn.get());
    }
}
```

```java
  File getFile() {
    return LogSegmentStartEnd.valueOf(startIndex, endIndex, isOpen).getFile(storage);
  }
```



## 4. LogSegment

```
In-memory cache for a log segment file.用来表示一个segmentFile在内存中的镜像
All the updates will be first written into LogSegment then into corresponding files in the same order.
```

注意这里的第二句话，所有的update都先在内存中更新，然后刷盘。LogSegment使用一个List存储LogRecord，即元数据信息，用来保证写入的有序性；使用一个\<TermIndex, LogEntryProto>的map，用来存储数据和检索。

这里先介绍其实例变量

```java
private volatile boolean isOpen;
private long totalFileSize = SegmentedRaftLogFormat.getHeaderLength(); //并没有使用final修饰
private AtomicLong totalCacheSize = new AtomicLong(0);
private final AtomicInteger loadingTimes = new AtomicInteger();

private long startIndex;
private volatile long endIndex;
private RaftStorage storage;
private final LogEntryLoader cacheLoader;

private final List<LogRecord> records = new ArrayList<>();
private final Map<TermIndex, LogEntryProto> entryCache = new ConcurrentHashMap<>();
```

首先重点理解以下getFile()实例方法，LogSegmentedStartEnd是value-based class，所以其内部实例变量的状态不能改变，那么当LogSegment的endIndex/isOpen状态发生改变时，都需要重新构造一个LogSegmentStartEnd对象，通过调用该对象的getFile(RaftStorage)方法返回对应的SegmentedRaftLog文件

```java
File getFile() {
    return LogSegmentStartEnd.valueOf(startIndex, endIndex, isOpen).getFile(storage);
}
```

numOfEntries()用来返回当前LogSegment中存在多少个LogEntry，这里需要关联到startIndex和endIndex的涵义是指LogSegment中缓存的第一个LogEntry的Index和最后一个LogEntry的index。要从FileSystem中的buffer/cache概念去理解LogSegment，LogSegment指的是向OS申请一块内存用来缓存disk中的内容，它并没有跟某一块block绑定，可以动态的更换其缓存的内容，所以这也是LogSegment中实例变量(startIndex/endIndex/isOpen)没有使用final修饰的原因。

接下来看实例方法append(boolean, LogEntryProto, Op)方法，这里是无论如何，都会按序记载LogEntry的元数据信息，即构造一个LogRecord对象并将其append到records这个list中，根据keepEntryInCache这个标志位，确定是否将其放到map中缓存

```java
private void append(boolean keepEntryInCache, LogEntryProto entry, Op op) {
    final LogRecord record = new LogRecord(totaFileSize, entry);
    records.add(record);
    
    if(keepEntryInCache) {
        putEntryCache(record.getTermIndex(), entry, op);
    }
    
    totalFileSize += getEntrySize(entry, op);
    entryIndex = entry.getIndex();
}
```

这里涉及到两个方法，分别为实例方法putEntryCache(TermIndex, LogEntryProto, Op)静态方法getEntrySize(LogEntyProto, Op)

```java
void putEntryCache(TermIndex key, LogEntryProto value, Op op) {
    final LogEntryProto previous = entryCache.put(key, value);
    long previousSize = 0;
    if(previous != null) {
        previousSize = getEntrySize(value, Op.REMOVE_CACHE);
    }
    totalCacheSize.getAndAdd(getEntrySize(value, op) - previousSize);
}
```

其几个静态方法也比较重要

首先是newOpenSegment() newClosedSegment()还有newLogSegment()方法，其中，判断open还是closed是通过其实例变量isOpen标志位以及endIndex的大小，当以open的方式打开Segment时，endIndex=startIndex-1

接下来是readSegmentFile方法，这个方法就是构造一个输入流对象，即SegmentedRaftLogInputStream对象，遍历调用其nextEntry()方法，返回读到的entry的数量。这里的核心主要是entryConsumer，即读到entry之后的处理逻辑在实参中，以调用方静态方法loadSegment()为例

```java
  public static int readSegmentFile(File file, LogSegmentStartEnd startEnd,
      CorruptionPolicy corruptionPolicy, SegmentedRaftLogMetrics raftLogMetrics, Consumer<LogEntryProto> entryConsumer)
      throws IOException {
      
  }
```

```java
  static LogSegment loadSegment(RaftStorage storage, File file, LogSegmentStartEnd startEnd,
      boolean keepEntryInCache, Consumer<LogEntryProto> logConsumer, SegmentedRaftLogMetrics raftLogMetrics)
      throws IOException {
    //构造一个返回的LogSegment对象，通过startEnd对象来确定其open/close状态
    final LogSegment segment = newLogSegment(storage, startEnd, raftLogMetrics);
    final CorruptionPolicy corruptionPolicy = CorruptionPolicy.get(storage, RaftStorage::getLogCorruptionPolicy);
    final boolean isOpen = startEnd.isOpen();
    //调用readSegmentFile()方法
    final int entryCount = readSegmentFile(file, startEnd, corruptionPolicy, raftLogMetrics, entry -> {
      segment.append(keepEntryInCache || isOpen, entry, Op.LOAD_SEGMENT_FILE);
      if (logConsumer != null) {
        logConsumer.accept(entry);
      }
    });
    LOG.info("Successfully read {} entries from segment file {}", entryCount, file);

    final long start = startEnd.getStartIndex();
    final long end = isOpen? segment.getEndIndex(): startEnd.getEndIndex();
    final int expectedEntryCount = Math.toIntExact(end - start + 1);
    final boolean corrupted = entryCount != expectedEntryCount;
    if (corrupted) {
      LOG.warn("Segment file is corrupted: expected to have {} entries but only {} entries read successfully",
          expectedEntryCount, entryCount);
    }

    if (entryCount == 0) {
      // The segment does not have any entries, delete the file.
      FileUtils.deleteFile(file);
      return null;
    } else if (file.length() > segment.getTotalFileSize()) {
      // The segment has extra padding, truncate it.
      FileUtils.truncateFile(file, segment.getTotalFileSize());
    }

    try {
      segment.assertSegment(start, entryCount, corrupted, end);
    } catch (Exception e) {
      throw new IllegalStateException("Failed to read segment file " + file, e);
    }
    return segment;
  }
```

