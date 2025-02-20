## 1. 前言

Client侧的RPC接口定义，用来发送数据，目前只有一个实现类NettyClientStreamRpc

## 2. 接口定义

```java
public interface DataStreamClientRpc extends Closeable {
  /** Async call to send a request. */
  default CompletableFuture<DataStreamReply> streamAsync(DataStreamRequest request) {
    throw new UnsupportedOperationException(getClass() + " does not support "
        + JavaUtils.getCurrentStackTraceElement().getMethodName());
  }
}
```

## 3. NettyClientStreamRpc实现类

发送逻辑在streamAsync(DataStreamRequest)方法中；输入流的回调函数正在getClientHandler()方法中，用来监控服务端发回的ack

