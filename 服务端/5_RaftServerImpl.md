## 1. 前言

Raft集群在收到client的请求后，调用RaftServerImpl.submitClientRequestAsync(RaftClientRequest)方法，此时我们处在leader端，那么接下来，我们以此方法为切入点，查看整个流程。在这里，我们以写操作为例，代码如下

```java
TransactionContext context = 		   				     
    stateMachine.startTransaction(filterDataStreamRaftClientRequest(request));
if (context.getException() != null) {
    final StateMachineException e = new StateMachineException(getMemberId(), context.getException());
    final RaftClientReply exceptionReply = newExceptionReply(request, e);
    cacheEntry.failWithReply(exceptionReply);
    replyFuture =  CompletableFuture.completedFuture(exceptionReply);
} else {
    replyFuture = appendTransaction(request, context, cacheEntry);
}
```

