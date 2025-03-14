## 1. 前言

这个类是一个线程类，继承自Runnable，持有一个实例变量Thread updater，在构造器中赋值updater=new Daemon(this)。用于跟踪已经committed的日志，并将日志应用于stateMachine中

入口方法在start()方法中

```java
void start() {
    initializeMetrics();
    updater.start();
}
```

updater.start()即调用StateMachineUpdater.run()方法。

```java
  public void run() {
    for(; state != State.STOP; ) {
      try {
        waitForCommit();

        if (state == State.RELOAD) {
          reload();
        }

        final MemoizedSupplier<List<CompletableFuture<Message>>> futures = applyLog();
        checkAndTakeSnapshot(futures);

        if (shouldStop()) {
          checkAndTakeSnapshot(futures);
          stop();
        }
      } catch (Throwable t) {
        if (t instanceof InterruptedException && state == State.STOP) {
          LOG.info("{} was interrupted.  Exiting ...", this);
        } else {
          state = State.EXCEPTION;
          LOG.error(this + " caught a Throwable.", t);
          server.close();
        }
      }
    }
  }
```

乍一看，逻辑很简单，调用waitForCommit()方法等带有新的committed的logEntry，然后调用applyLog()将这部分日志应用到stateMachine中

```java
  private void waitForCommit() throws InterruptedException {
    // When a peer starts, the committed is initialized to 0.
    // It will be updated only after the leader contacts other peers.
    // Thus it is possible to have applied > committed initially.
    final long applied = getLastAppliedIndex();
    for(; applied >= raftLog.getLastCommittedIndex() && state == State.RUNNING && !shouldStop(); ) {
      if (awaitForSignal.await(100, TimeUnit.MILLISECONDS)) {
        return;
      }
    }
  }
```

接下来就是applyLog()的逻辑，这里注意applyLog的返回值，是一个List<CompletableFuture\<Message>>对象，这是因为RaftServerImpl.applyLogToStateMahcine()方法返回的是一个CompletableFuture\<Message>对象，而applyLog方法针对的是appliedIndex到committedIndex之间的logEntries，所以这里返回一个list

```java
private MemoizedSupplier<List<CompletableFuture<Message>>> applyLog() throws RaftLogIOException {
    final MemoizedSupplier<List<CompletableFuture<Message>>> futures = MemoizedSupplier.valueOf(ArrayList::new);
    final long committed = raftLog.getLastCommittedIndex();
    for(long applied; (applied = getLastAppliedIndex()) < committed && state == State.RUNNING && !shouldStop(); ) {
        final long nextIndex = applied + 1;
        //这里是从raftLog对象中取出下一条commit的logEntry，给到stateMachine去apply
        final LogEntryProto next = raftLog.get(nextIndex);
        if (next != null) {
            if (LOG.isTraceEnabled()) {
                LOG.trace("{}: applying nextIndex={}, nextLog={}", this, nextIndex, LogProtoUtils.toLogEntryString(next));
            } else {
                LOG.debug("{}: applying nextIndex={}", this, nextIndex);
            }

            final CompletableFuture<Message> f = server.applyLogToStateMachine(next);
            if (f != null) {
                futures.get().add(f);
            }
            final long incremented = appliedIndex.incrementAndGet(debugIndexChange);
            Preconditions.assertTrue(incremented == nextIndex);
        } else {
            LOG.debug("{}: logEntry {} is null. There may be snapshot to load. state:{}",
                      this, nextIndex, state);
            break;
        }
    }
    return futures;
}
```

因为applyLog方法的返回值，实际上是一个List\<CompletableFuture\<Message>>对象，这里将这个返回值传递给checkAndTakeSnapshot方法，由其来决定是否触发takeSnapshot动作

```java
private void checkAndTakeSnapshot(MemoizedSupplier<List<CompletableFuture<Message>>> future) throws ExecutionException, InterruptedException {
    if(shouldTakeSnapshot()) {
        if(futures.isInitialized()) {
            JavaUtils.allOf(futures.get()).get();
        }
        
        takeSnapshot();
    } 
}

private boolean shouldTakeSnapshot() {
    //这个条件是后加的，先不看
    if(state == State.RUNNING && server.getSnapshotRequestHandler().shouldTriggerSnapshot(())) {
        return true;
    }
       
    if(autoSnapshotThreshold == null) {
        return false;
    } else if(shouldStop()) {
        return getLastAppliedIndex() - snapshotIndex.get() > 0;
    }   
    
    return state == State.RUNNING && 
        getStateMachineLastAppliedIndex() - snapshotIndex.get() >= autoSnapshotThreshold;
       
}
```

