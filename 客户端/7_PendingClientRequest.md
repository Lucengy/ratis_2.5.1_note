## 1. 前言

抽象类，用来标识来自客户端的请求

## 2. 代码注释

实际上就是对RaftClientRequest的封装，主要记录了一些统计信息，以及提供了

* RaftClientRequest newRequestImpl()

方法

```java
  public abstract static class PendingClientRequest {
    private final long creationTimeInMs = System.currentTimeMillis();
    private final CompletableFuture<RaftClientReply> replyFuture = new CompletableFuture<>();
    private final AtomicInteger attemptCount = new AtomicInteger();
    private final Map<Class<?>, Integer> exceptionCount = new ConcurrentHashMap<>();

    public abstract RaftClientRequest newRequestImpl();

    final RaftClientRequest newRequest() {
      attemptCount.incrementAndGet();
      return newRequestImpl();
    }

    CompletableFuture<RaftClientReply> getReplyFuture() {
      return replyFuture;
    }

    public int getAttemptCount() {
      return attemptCount.get();
    }

    int incrementExceptionCount(Throwable t) {
      return t != null ? exceptionCount.compute(t.getClass(), (k, v) -> v != null ? v + 1 : 1) : 0;
    }

    public int getExceptionCount(Throwable t) {
      return t != null ? Optional.ofNullable(exceptionCount.get(t.getClass())).orElse(0) : 0;
    }

    public boolean isRequestTimeout(TimeDuration timeout) {
      if (timeout == null) {
        return false;
      }
      return System.currentTimeMillis() - creationTimeInMs > timeout.toLong(TimeUnit.MILLISECONDS);
    }
  }
```

## 3. 实现类PendingOrderedRequest

OrderedAsyn中的静态内部类

这里的requestConstructor类型看着很唬人，这里的SlidingWindowEntry并不是SlidingWindow中定义的相关类，而是protobuf中的消息类型，有关其定义说明如下

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

RaftRpcRequestProto是发起RPC时携带的元数据信息，其中包含了SlidingWindowEntry，SlidingWindowEntry只包含了两个字段，分别是

* seqNum
* isFirst

用来标识这个rpc segement的seqNum，以及是否是整个segement的第一个请求，当seqNum为0时，代表这并不是滑动窗口的请求

```java
static class PendingOrderedRequest extends PendingClientRequest implements SlidingWindow.ClientSideRequest<RaftClientReply> {
    private final long callId;
    private final long seqNum;
    private final AtomicReference<Function<SlidingWindowEntry, RaftClientRequest>> requestConstructor;
    
    private volatile boolean isFirst;
    
    PendingOrderedRequest(long callId, long seqNum, Function<SlidingWindowEntry, RaftClientRequest> requestConstructor) {
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
    
    @Override
    public void setFirstRequest() {
        isFirst = true;
    }
    
    @Override
    public long getSeqNum() {
        return seqNum;
    }
    
    @Override
    public boolean hasReply() {
        return getReplyFuture().isDone();
    }
    
    @Override
    public void setReply(RaftClientReply reply) {
        requestConstructor.set(null);
        getReplyFuture().complete(reply);
    }
    
    @Override
    public void fail(Throwable e) {
        requestConstructor.set(null);
        getReplyFuture().completeExceptionally(e);
    }
}
```

看一下caller是怎么调用构造器的

```java
new PendingOrderedRequest(callId, seqNum,
        slidingWindowEntry -> client.newRaftClientRequest(server, callId, message, type, slidingWindowEntry));
```



