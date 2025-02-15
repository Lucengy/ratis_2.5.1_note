## 1. 客户端

proto文件定义如下: 

```protobuf
service RaftServerProtocolService {
  rpc requestVote(ratis.common.RequestVoteRequestProto)
      returns(ratis.common.RequestVoteReplyProto) {}

  rpc startLeaderElection(ratis.common.StartLeaderElectionRequestProto)
      returns(ratis.common.StartLeaderElectionReplyProto) {}

  rpc appendEntries(stream ratis.common.AppendEntriesRequestProto)
      returns(stream ratis.common.AppendEntriesReplyProto) {}

  rpc installSnapshot(stream ratis.common.InstallSnapshotRequestProto)
      returns(stream ratis.common.InstallSnapshotReplyProto) {}
}
```

RPC客户端类为GrpcServerProtocolClient，持有同步和异步两个stub对象

```java
  private final RaftServerProtocolServiceStub asyncStub;
  private final RaftServerProtocolServiceBlockingStub blockingStub;
```

rpc调用为installSnapshot()方法，该rpc为双向stream方法，故返回值StreamObserver\<InstallSnapshotRequestProto>用来监测输出流，而提供给rpc的StreamObserver\<InstallSnapshotReplyProto>用来检测输入流

```java
  StreamObserver<InstallSnapshotRequestProto> installSnapshot(
      StreamObserver<InstallSnapshotReplyProto> responseHandler) {
    return asyncStub.withDeadlineAfter(requestTimeoutDuration.getDuration(), requestTimeoutDuration.getUnit())
        .installSnapshot(responseHandler);
  }
```

该方法的调用方在GrpcLogAppender.installSnapshot(SnapshotInfo)方法中，输入流的StreamObserver对象为InstallSnapshotResponseHandler，该对象继承自StreamObserver\<InstallSnapshotReplyProto>

```java

```

![1739070551400](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1739070551400.png)



## 2. 服务端

RPC实现类为GrpcServerProtocolService，其installSnapshot()方法定义如下

```java
  @Override
  public StreamObserver<InstallSnapshotRequestProto> installSnapshot(
      StreamObserver<InstallSnapshotReplyProto> responseObserver) {
    return new ServerRequestStreamObserver<InstallSnapshotRequestProto, InstallSnapshotReplyProto>(
        RaftServerProtocol.Op.INSTALL_SNAPSHOT, responseObserver) {
      @Override
      CompletableFuture<InstallSnapshotReplyProto> process(InstallSnapshotRequestProto request) throws IOException {
        return CompletableFuture.completedFuture(server.installSnapshot(request));
      }

      @Override
      long getCallId(InstallSnapshotRequestProto request) {
        return request.getServerRequest().getCallId();
      }

      @Override
      String requestToString(InstallSnapshotRequestProto request) {
        return ServerStringUtils.toInstallSnapshotRequestString(request);
      }

      @Override
      String replyToString(InstallSnapshotReplyProto reply) {
        return ServerStringUtils.toInstallSnapshotReplyString(reply);
      }

      @Override
      boolean replyInOrder(InstallSnapshotRequestProto installSnapshotRequestProto) {
        return true;
      }
    };
  }
```

可以看到，该方法返回StreamObserver\<InstallSnapshotRequestProto>，用来监测输入流，形参类型为StreamObserver\<InstallSnapshotReplyProto>，用来监测输出流，是rpc本身提供的streamObserver对象。接下来将目光移至ServerRequestStreamObserver类中

#### 1. ServerRequestStreamObserver

泛型类，使用两个泛型REQUEST和REPLY，分别代表请求和响应，同时继承自StreamObserver\<REQUEST>

```java
  abstract class ServerRequestStreamObserver<REQUEST, REPLY> implements StreamObserver<REQUEST> 
```

持有一个StreamObserver\<REPLY>对象，因为这个类是给rpc的server端使用的，所以SeverRequestStreamObserver本身用来监测输入流，持有一个用来监测输出流的StreamObserver对象。

```java
private final StreamObserver<REPLY> responseObserver;
```

先讲以下PendingServerRequest类，只是对REQUEST进行了封装，同时持有了一个CompletableFuture\<Void>对象用来标识REQUEST是否完成，是GrpcServerProtocolService的静态内部类

```java
  static class PendingServerRequest<REQUEST> {
    private final REQUEST request;
    private final CompletableFuture<Void> future = new CompletableFuture<>();

    PendingServerRequest(REQUEST request) {
      this.request = request;
    }

    REQUEST getRequest() {
      return request;
    }

    CompletableFuture<Void> getFuture() {
      return future;
    }
  }
```

ServerRequestStreamObserver持有一个PendingServerRequest对象，用来记录最后一个请求以及其完成状态

```java
    private final AtomicReference<PendingServerRequest<REQUEST>> previousOnNext = new AtomicReference<>();
```

构造器就是简单的对responseObserver进行赋值

```java
    ServerRequestStreamObserver(RaftServer.Op op, StreamObserver<REPLY> responseObserver) {
      this.op = op;
      this.responseObserver = responseObserver;
    }
```

因为其继承自StreamObserver\<REQUEST>，所以重点关注其onNext() onComplete()和onError()方法，先看onNext()方法，这里需要关注的有两个方法，分别为process(REQUEST)和replyInOrder(REQUEST)方法

```java
 public void onNext(REQUEST request) {
      if (!replyInOrder(request)) {
        try {
          process(request).thenAccept(this::handleReply);
        } catch (Exception e) {
          handleError(e, request);
        }
        return;
      }

      final PendingServerRequest<REQUEST> current = new PendingServerRequest<>(request);
      final PendingServerRequest<REQUEST> previous = previousOnNext.getAndSet(current);
      final CompletableFuture<Void> previousFuture = Optional.ofNullable(previous)
          .map(PendingServerRequest::getFuture)
          .orElse(CompletableFuture.completedFuture(null));
      try {
        process(request).exceptionally(e -> {
          // Handle cases, such as RaftServer is paused
          handleError(e, request);
          current.getFuture().completeExceptionally(e);
          return null;
        }).thenCombine(previousFuture, (reply, v) -> {
          handleReply(reply);
          current.getFuture().complete(null);
          return null;
        });
      } catch (Exception e) {
        handleError(e, request);
        current.getFuture().completeExceptionally(e);
      }
    }
```

