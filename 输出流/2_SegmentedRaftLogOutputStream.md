## 1. 前言

RaftLog按如下格式进行存储

![1736955303186](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1736955303186.png)

需要关注的一点是，serialized size of the entry是entry的size，这是一个int值，占用4个字节。在存储时，使用了protobuf对该int值进行编码，其占用1-5个字节，但是在原始int值较小时，其序列化后占用的字节数越少，使用protobuf编码存储int值可以有效的减少存储空间

首先，需要关注的一点是构造器，RaftLog支持append写。在createNewRaftLog的情况下，需要将RaftLogHeader即"RaftLog1".getBytes(StandardCharsets.UTF_8)写入到RaftLog文件中；但是在append的情况下，RaftLog文件已经存在，那么RaftLogHeader已经写入到RaftLog文件中了，就不再需要提前写入RaftLog中。这部分的逻辑体现在构造器中

```java
if(!append) {
    preallocateIfNecessary(SegmentedRaftLogFormat.getHeaderLength());
    SegmentedRaftLogFormat.appendHeader(CheckedConsumber.asCheckedFunction(out::write));
    out.flush();
}
```

三行代码三个逻辑

1. 预分配空间，预分配的大小是RaftLogHeader的大小，即"RaftLog1".getBytes(StandardCharsets.UTF_8) 4字节
2. 调用BufferedWriteChannel.write(byte[])方法，将RaftLogHeader写入到其持有的实例变量ByteBuffer缓存中
3. 调用BufferedWriteChannel.flush()方法，将刚刚写入的RaftLogHeader信息刷盘

如果是append模式打开的RaftLog，那么就不存在上述逻辑

接下来重中之重的方法就是write(LogEntryProto)方法，代码中的JavaDoc如下

```java
/**
write the given entry to this output stream.
Format:
* (1) The serialized size of the entry
* (2) The entry
* (3) 4-byte checksum of the entry.

Size in bytes to be writen:
 * (size to encode n) + n + (checksum size)
 * where n is the entry serialized size and the checksum size is 4.
```

方法如下

```java
public void write(LogEntryProto entry) throws IOException {
    final int serialized = entry.getSerializedSize(); //这里是int类型
    //序列化后size的大小+entry大小
    final int proto = CodedOutputStream.computeUInt32SizeNoTag(serialized) + serialized;
    final byte[] buf = new byte[proto + 4]; //加上checksum的大小，这里就是一条logEntry的实际落盘大小
    preallocateIfNecessary(buf.length);
    
    CodedOutputStream cout = CodedOutputStream.newInstance(buf);
    cout.writeUInt32NoTag(serialized); //以protobuf格式序列化写入entry size到buf中
    entry.writeTo(cout); //将entry写入buf中
    
    checksum.reset();
    checksum.update(buf, 0, proto);
    ByteBuffer.wrap(buf, proto, 4).putInt((int) checksum.getValue()); //将checksum写入到buf中
    out.write(buf); //调用BufferedWriteChannel.write(byte[])方法写入整条logEntry信息
}
```

剩下需要理解的方法是actualPreallocateSize(long, long, long)方法。首先看想要写入的数据大小，若像要写入的数据>剩余空间，返回想要写入的数据大小；否则，如果想要写入的数据>预分配的空间，返回想要写入数据的大小；否则，返回剩余空间和预分配大小二者的小值

```java
  private static long actualPreallocateSize(long outstandingData, long remainingSpace, long preallocate) {
    return outstandingData > remainingSpace? outstandingData
        : outstandingData > preallocate? outstandingData
        : Math.min(preallocate, remainingSpace);
  }
```

其使用private修饰，调用方在preallocate(FileChannel, long)

```java
  private long preallocate(FileChannel fc, long outstanding) throws IOException {
    final long actual = actualPreallocateSize(outstanding, segmentMaxSize - fc.size(), preallocatedSize);
    Preconditions.assertTrue(actual >= outstanding);
    final long allocated = IOUtils.preallocate(fc, actual, FILL);
    LOG.debug("Pre-allocated {} bytes for {}", allocated, this);
    return allocated;
  }
```

