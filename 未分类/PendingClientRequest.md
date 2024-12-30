要想理解滑动窗口的工作模式，先要理解PendingClientRequest

PendingRequest只有两个字段需要知道一下

* replyFuture
* attemptCount

```java
    private final CompletableFuture<RaftClientReply> replyFuture = new CompletableFuture<>();
    private final AtomicInteger attemptCount = new AtomicInteger();
	
    final RaftClientRequest newRequest() {
      attemptCount.incrementAndGet();
      return newRequestImpl();
    }

    public abstract RaftClientRequest newRequestImpl();
```

replyFuture是在类构造时就构造好了已经，最终调用replyFuture.complete()方法赋值即可

attempCount是在构造器中执行自增操作的，这是在retry request时进行的

newRequestImpl是一个抽象方法，由子类实现，难度在ordered，那么我们关注其PendingOrderedRequest

## PendingOrderedRequest

属于OrderedAsync的静态内部类，继承自PendingClientRequest，实现了 SlidingWindow.ClientSideRequest\<RaftClientReply>接口

关于字段信息

* callId 主要用来打印日志信息
* seqNum 请求ID，主要给ordered RPC使用的，即SlidingWindow里面要用到的
* isFirst 同上
* requestConstructor AtomicReference\<Function\<SlidingWindowEntry, RaftClientRequest>> 类型，函数式调用接口

其实在Raft.proto文件中就很清晰明了的知道上述字段的用处

```protobuf
message RaftRpcRequestProto {
  bytes requestorId = 1;
  bytes replyId = 2;
  RaftGroupIdProto raftGroupId = 3;
  uint64 callId = 4;
  bool toLeader = 5;

  uint64 timeoutMs = 13;
  RoutingTableProto routingTable = 14;
  SlidingWindowEntry slidingWindowEntry = 15;
}

message SlidingWindowEntry {
  uint64 seqNum = 1; // 0 for non-sliding-window requests; >= 1 for sliding-window requests
  bool isFirst = 2;  // Is this the first request of the sliding window?
}
```

注释也是源代码中的注释，seqNum和isFirst是给SlidingWindowEntry使用的，SlidingWindowEntry是给RaftRpcRequestProto使用的，后者是ordered和unordered RPC使用的参数

该类值得关注的只有其实现的抽象方法-->newRequestImpl

```java
    PendingOrderedRequest(long callId, long seqNum,
        Function<SlidingWindowEntry, RaftClientRequest> requestConstructor) {
      this.callId = callId;
      this.seqNum = seqNum;
      this.requestConstructor = new AtomicReference<>(requestConstructor);
    }

    @Override
    public RaftClientRequest newRequestImpl() {
      return Optional.ofNullable(requestConstructor.get())
          .map(f -> f.apply(ProtoUtils.toSlidingWindowEntry(seqNum, isFirst)))
          .orElse(null);
    }
```

注意requestConstructor是根据SlidingWindowEntry来构造RaftClientRequest的，说明构造RaftClientRequest的核心内容都在这里了，其实应该是对已经构建好的RaftClientRequestProto添加SlidingWindowEntry信息。

还有一点需要说明的是，isFirst默认设置为False，只有在调用setFirstRequest()方法时，才会将该对象的isFirst状态设置为true