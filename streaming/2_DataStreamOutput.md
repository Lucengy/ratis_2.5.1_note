## 1. 前言

如JavaDoc所说，是一个异步的输出流对象，主要包含writeAsync()相关方法

```java
public interface DataStreamOutput extends CloseAsync<DataStreamReply> {
  /**
   * Send out the data in the source buffer asynchronously.
   *
   * @param src the source buffer to be sent.
   * @param options - options specifying how the data was written
   * @return a future of the reply.
   */
  CompletableFuture<DataStreamReply> writeAsync(ByteBuffer src, WriteOption... options);


  /**
   * The same as writeAsync(src, 0, src.length(), sync_default).
   * where sync_default depends on the underlying implementation.
   */
  default CompletableFuture<DataStreamReply> writeAsync(File src) {
    return writeAsync(src, 0, src.length());
  }

  /**
   * The same as writeAsync(FilePositionCount.valueOf(src, position, count), options).
   */
  default CompletableFuture<DataStreamReply> writeAsync(File src, long position, long count, WriteOption... options) {
    return writeAsync(FilePositionCount.valueOf(src, position, count), options);
  }

  /**
   * Send out the data in the source file asynchronously.
   *
   * @param src the source file with the starting position and the number of bytes.
   * @param options options specifying how the data was written
   * @return a future of the reply.
   */
  CompletableFuture<DataStreamReply> writeAsync(FilePositionCount src, WriteOption... options);

  /**
   * Return the future of the {@link RaftClientReply}
   * which will be received once this stream has been closed successfully.
   * Note that this method does not trigger closing this stream.
   *
   * @return the future of the {@link RaftClientReply}.
   */
  CompletableFuture<RaftClientReply> getRaftClientReplyFuture();

  /**
   * @return a {@link WritableByteChannel} view of this {@link DataStreamOutput}.
   */
  WritableByteChannel getWritableByteChannel();
}
```

