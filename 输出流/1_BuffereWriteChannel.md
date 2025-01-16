## 1. 前言

持有四实例变量，分别为

* FileChannel fileChannel
* ByteBuffer writeBuffer
* Boolean forced = true
* final Supplier<CompletableFuture\<Void>> flushFuture

其中，fileChannel和writeBuffer是主要操作的对象，这里需要理解的实例变量是forced，首先要理解数据写入流程

1. 数据写入到writeBuffer中
2. 通过调用fileChannel.write(BybteBuffer)方法将数据写入文件中
3. 第2部写入的数据并没有真正落盘，而是在OS的buffer/cache中，需要调用FileChannel.force(boolean)方法将数据刷新到磁盘中，形参中的boolean值指定是否将metadata也刷新到磁盘上，即inode相关信息

第2部调用的是实例方法flushBuffer()方法，将Bytebuffer中的数据写到OS的buffer/cache中，此时有数据写入，但是还没有同步，需要将forced置位false，在flush()方法和asyncFlush(ExecutorService)中对forced变量进行判断

* 若forced为false，证明有数据已经写入到OS的buffer/cache中，但是还没有刷盘，那么调用fileChannel.force()方法，并将forced置为true
* 若force的为true，证明重复调用了flush()方法，并没有数据写入到OS中，那么直接跳过即可。

这里需要关注的方法有两个，分别为

* write(byte[])
* preallocateIfNecessary(long, CheckedBiFunction\<FileChannel, Long, Long, IOExcepiton>)

```java
void write(byte[] b) throws IOException {
    int offset = 0;
    while(offset < b.length) {
        int toPut = Math.min(b.length - offset, writeBuffer.reamining());
        writeBuffer.put(b, offset, toPut);
        offset += toPut;
        if(writeBuffer.remaining() == 0) {
            flushBuffer();
        }
    }
}
```

write(byte[])方法，是将数据接入到ByteBuffer中，若ByteBuffer写满了，则调用flushBuffer()方法将ByteBuffer中的数据写到OS的buffer/cache中，这里并没有调用FileChannel.sync()方法，换言之，并没有强制刷盘

```java
void preallocateIfNececessary(long size, CheckedBiFunction<FileChannel, Long, Long, IOException> preallocate) throws IOException{
    final long outstanding = writeBuffer.position() + size;
    if(fileChannel.positon() + outstanding > fileChannel.size()) {
        preallocate.apply(fileChannel, outstanding);
    }
}
```

