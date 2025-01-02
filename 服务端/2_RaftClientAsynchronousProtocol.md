接口定义，是Raft leader用来处理client的入口方法

## 1. 接口定义

目前知道这个是异步的就好了，果然有异步，就有同步，同步接口定义为RaftClientProtocol

```java
public interface RaftClientAsynchronousProtocol {
  CompletableFuture<RaftClientReply> submitClientRequestAsync(
      RaftClientRequest request) throws IOException;
}
```

```java
public interface RaftClientProtocol {
  RaftClientReply submitClientRequest(RaftClientRequest request) throws IOException;
}
```

