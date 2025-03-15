## 0. 前言

这里有关snapshot有三部分

1. stateMachine会通过takeSnapshot动作做log的compaction，这部分的切入点为StateMachine.takeSnapshot()方法，调用方为StateMachineUpdater.run()方法，循环调用checkAndTakeSnapshot()方法
2. leader发送installSnapshot rpc给follower
3. leader发送notify rpc给follower

逻辑上先看sateMachine.takeSnapshot方法，而后看rpc的过程



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

该方法的调用方在GrpcLogAppender.installSnapshot(SnapshotInfo)方法中，输入流的StreamObserver对象为InstallSnapshotResponseHandler，该对象继承自StreamObserver\<InstallSnapshotReplyProto>。

罪恶的根源为Ratis-498，一开始installSnapshot的涉及为：出现怠速的follower，当leader的第一条raftLog大于follower最后一条raftLog时，leader应该发送一个installSnapshotRequest到follower，这也是raft原文描述的场景

好死不死，这里要加一个场景，就是将stateMachine的installSnapshot从ratis server中解耦出来。那么出现上述问题时，leader只是给follower发送一个notify的rpc，而不是由logAppender将installSnapshot发送给follower，follower再收到notify  rpc时，notify自己的stateMachine，告知stateMachine需要从leader搞一个snapshot过来

```
When a lagging Follower wants to catch up with the Leader, and the Leader only has logs with start index greater than the Followers's last log index, then the leader sends an InstallSnapshotRequest to the the Follower. 

The aim of this Jira is to allow State Machine to decouple snapshot installation from the Ratis server. When Leader does not have the logs to get the Follower up to speed, it should notify the Follower to install a snapshot (if install snapshot through Log Appender is disabled). The Follower in turn notifies its state machine that a snapshot is required to catch up with the leader.
```

上述两种情况是互斥的，要么是leader主动发起installSnapshotRequest，要么是leader只是发送notify rpc，由follower去拉去snapshot。这部分的逻辑在GrpcLogAppender.run()方法中有所体现。installSnapshot(SnapshotInfo)和installSnapshot(TermIndex)对应两种rpc方式，在if else中是互斥的

```java
if (installSnapshotEnabled) {
    SnapshotInfo snapshot = shouldInstallSnapshot();
    if (snapshot != null) {
        installSnapshot(snapshot);
        installSnapshotRequired = true;
    }
} else {
    TermIndex installSnapshotNotificationTermIndex = shouldNotifyToInstallSnapshot();
    if (installSnapshotNotificationTermIndex != null) {
        installSnapshot(installSnapshotNotificationTermIndex);
        installSnapshotRequired = true;
    }
}
```

紧接着，就是shouldInstallSnapshot()方法和shouldNotifyToInstallSnapshot()方法，这两个方法对应着逻辑，按着ratis498所描述的，前者跟Raft server强相关，后者是跟raft server解耦的，那么看一下这两个方法的逻辑

shouldInstallSnapshot()方法是LogAppender中的default方法，根据JavaDoc，存在三种情况

* leader没有任何logEntry
* follower's next index小于leader的startIndex
* follower刚刚启动

```java
default SnapshotInfo shouldInstallSnapshot() {
    // we should install snapshot if the follower needs to catch up and:
    // 1. there is no local log entry but there is snapshot
    // 2. or the follower's next index is smaller than the log start index
    // 3. or the follower is bootstrapping and has not installed any snapshot yet
    final FollowerInfo follower = getFollower();
    final boolean isFollowerBootstrapping = getLeaderState().isFollowerBootstrapping(follower);
    final SnapshotInfo snapshot = getServer().getStateMachine().getLatestSnapshot();

    if (isFollowerBootstrapping && !follower.hasAttemptedToInstallSnapshot()) {
        if (snapshot == null) {
            // Leader cannot send null snapshot to follower. Hence, acknowledge InstallSnapshot attempt (even though it
            // was not attempted) so that follower can come out of staging state after appending log entries.
            follower.setAttemptedToInstallSnapshot();
        } else {
            return snapshot;
        }
    }

    final long followerNextIndex = getFollower().getNextIndex();
    if (followerNextIndex < getRaftLog().getNextIndex()) {
        final long logStartIndex = getRaftLog().getStartIndex();
        if (followerNextIndex < logStartIndex || (logStartIndex == RaftLog.INVALID_LOG_INDEX && snapshot != null)) {
            return snapshot;
        }
    }
    return null;
}
```



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

