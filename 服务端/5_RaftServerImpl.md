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

appendTransaction(request, context, cacheEntry)方法最终调用ServerState.appendLog(TransactionContext)方法

```java
  private CompletableFuture<RaftClientReply> appendTransaction(
      RaftClientRequest request, TransactionContext context, CacheEntry cacheEntry) throws IOException {
    assertLifeCycleState(LifeCycle.States.RUNNING);
    CompletableFuture<RaftClientReply> reply;

    final PendingRequest pending;
    synchronized (this) {
      reply = checkLeaderState(request, cacheEntry, true);
      if (reply != null) {
        return reply;
      }

      // append the message to its local log
      final LeaderStateImpl leaderState = role.getLeaderStateNonNull();
      final PendingRequests.Permit permit = leaderState.tryAcquirePendingRequest(request.getMessage());
      if (permit == null) {
        cacheEntry.failWithException(new ResourceUnavailableException(
            getMemberId() + ": Failed to acquire a pending write request for " + request));
        return cacheEntry.getReplyFuture();
      }
      try {
        state.appendLog(context);
      } catch (StateMachineException e) {
        // the StateMachineException is thrown by the SM in the preAppend stage.
        // Return the exception in a RaftClientReply.
        RaftClientReply exceptionReply = newExceptionReply(request, e);
        cacheEntry.failWithReply(exceptionReply);
        // leader will step down here
        if (e.leaderShouldStepDown() && getInfo().isLeader()) {
          leaderState.submitStepDownEvent(LeaderState.StepDownReason.STATE_MACHINE_EXCEPTION);
        }
        return CompletableFuture.completedFuture(exceptionReply);
      }

      // put the request into the pending queue
      pending = leaderState.addPendingRequest(permit, request, context);
      if (pending == null) {
        cacheEntry.failWithException(new ResourceUnavailableException(
            getMemberId() + ": Failed to add a pending write request for " + request));
        return cacheEntry.getReplyFuture();
      }
      leaderState.notifySenders();
    }
    return pending.getFuture();
  }
```



ServerState.appendLog(TransactionContext)方法调用了RaftLog.append(long, TransactionContext)方法，代码在RaftLogBase类中

```java
  @Override
  public final long append(long term, TransactionContext transaction) throws StateMachineException {
    return runner.runSequentially(() -> appendImpl(term, transaction));
  }

  private long appendImpl(long term, TransactionContext operation) throws StateMachineException {
    checkLogState();
    try(AutoCloseableLock writeLock = writeLock()) {
      final long nextIndex = getNextIndex();

      // This is called here to guarantee strict serialization of callback executions in case
      // the SM wants to attach a logic depending on ordered execution in the log commit order.
      try {
        operation = operation.preAppendTransaction();
      } catch (StateMachineException e) {
        throw e;
      } catch (IOException e) {
        throw new StateMachineException(memberId, e);
      }

      // build the log entry after calling the StateMachine
      final LogEntryProto e = operation.initLogEntry(term, nextIndex);

      int entrySize = e.getSerializedSize();
      if (entrySize > maxBufferSize) {
        throw new StateMachineException(memberId, new RaftLogIOException(
            "Log entry size " + entrySize + " exceeds the max buffer limit of " + maxBufferSize));
      }
      appendEntry(e);
      return nextIndex;
    }
  }
```

## 2. leader流程

接下来需要重点理解的就是Leader整个工作流程了。从RaftServerImpl.submitClientRequestAsync(RaftClientRequest)方法中，我们可以看到调用了

* StateMachine.startTransaction(RaftClientRequest)

紧接着，在RaftLogBase.appendImpl(long, Transaction)中调用了operation.preAppendTransaction()方法，这实际上是调用了

* StateMachine.preAppendTransaction()

在这里之后，是将logEntry放到leader的log中。回到RaftServerImpl.appendTransaction(request, context, cacheEntry)方法中，在调用ServerState.appendLog(TransactionContext)之后，调用LeaderState.notifySenders()，将对应的logEntry发送到各个Follower，然后由单独的线程StateMachineUpdater处理commited之后的logEntries，这部分的逻辑在StateMachineUpdater.run()方法中

StateMachineUpdater.run()方法调用applyLog()方法，循环调用applyLogToStateMachine(LogEntryProto)方法，在applyLogToStateMachine(LogEntryProto)方法中，按序调用了

* StateMachine.applyTransactionSerial(TransactionContext)
* StateMachine.applyTransaction(TransactionContext)

总结下来，流程如下:

* leader收到logEntry
* StateMachine.startTransaction(RaftClientRequest)
* StateMachine.preAppendTransaction()
* leader将logEntry放到自己的RaftLog中
* leader将logEntry发给各follower
* 针对已经committed的信息，leader调用StateMachine.applyTransacionSerial(TransactionContext)和StateMachine.applyTransaction(TransactionContext)方法

BackTo Ozone中的StateMachine实现类OzoneManagerStateMachine