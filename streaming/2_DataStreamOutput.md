## 1. 前言

根据RATIS-1085中的描述，在Streaming Pipeline中需要先发送一个header信息，即RaftClientRequest对象，以便server在收到该写请求时，能够正常处理，这就引发了一个新的问题，就是RaftClientReuqest对象究竟时怎么一个事，以及SM是怎么跟它进行交互的

```
In a stream request, the client should send a RaftClientRequest (without data) as the header so that the state machine at the server can process the request as a normal RaftClientRequest.

We may consider using Protobuf to encode RaftClientRequest. The raw data will be streamed after the RaftClientRequest.
```



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

## 2. DataStreamOutputRpc

继承自DataStreamOutput

```java
public interface DataStreamOutputRpc extends DataStreamOutput {
  /** Get the future of the header request. */
  CompletableFuture<DataStreamReply> getHeaderFuture();
}
```

## 3. DataStreamOutputImpl

实现类，用来处理输出流的逻辑。