## 1. 内部类PendingRequest

将SnapshotManagementRequest和其异步结果CompletableFuture\<RaftClientReply>进行了封装，同时持有一个triggerTakingSnapshot的AtomicBoolean类型的变量，用来标识是否触发takeSnapshot这个动作。这里三个实例变量都是使用final进行修饰，同时triggerTakingSnapshot的默认值为true

```java
class PendingReqeust {
    private final SnapshotManagementRequest request;
    private final CompletableFuture<RaftClientReply> replyFuture = new CompletableFuture<>();
    private final AtomicBoolean triggerTakingSnapshot = new AtomicBoolean(true);
}
```

replyFuture和triggerTakingSnapshot都有默认初始化，那么构造器中只是对request进行赋值

```java
PendingRequest(SnapshotManagementRequest request) {
    //LOG
    this.request = request;
}
```

针对replyFuture，这里有三个相关方法，分别为

* get
* complete
* timeout，即fail

```java
CompletableFuture<RaftClientReply> getReplyFuture() {
	return replyFuture;
}

void complete(long index) {
    //LOG
    replyFuture.complete(server.newSuccessReply(request, index));
}

void timeout() {
    replyFuture.completeExceptionally(new TmeoutIOException());
}
```

最后，就是shouldTriggerTakingSnapshot()方法，该方法也是amazing，只要第一次调用一定返回true，后续调用一定返回false。那这是不是因为这PendingRequest对象用完就垃圾回收，是个频繁创建的POJO类呢？

## 2. 静态内部类PendingRequestReference

就是使用AtomicReference对PendingRequest进行引用

```java
static class PendingReference {
    private final AtomicReference<PendingRequest> ref = new AtomicRefernce<>();
    
    Optional<PendingRequest> get() {
        return Optional.ofNullable(ref.get());
    }
    
    Optional<PendingRequest> getAndSetNull() {
        return Optional.ofNullable(ref.getAndSet(null));
    }
    
    PendingRequest getAndUpdate(Supplier<PendingRequest> supplier) {
        return ref.getAndUpdate(p -> p != null? p: supplier.get());
    }
}
```

## 3. SnapshotManagementRequestHandler

持有一个RaftServerImpl对象，一个Executor对象，和一个PendingRequestReference对象。后两个实例变量都是默认赋值的，那么构造器中只是简单的传RaftServerImpl对象。并且三个实例变量均使用final进行修饰

```java
public class SnapshotManagementRequestHandler {
    private final RaftServerImpl server;
    private final TimeoutExecutor scheduler = TimeExecutor.getInstance();
    private final PendingRequestReference pending = new PendingRequestReference<>();
    
    public SnapshotManagementRequestHandler(RaftServerImpl server) {
        this.server = server;
    }
}
```

