## 1. 前言

这个类是一个线程类，继承自Runnable，持有一个实例变量Thread updater，在构造器中赋值updater=new Thread(this)。用于跟踪已经committed的日志，并将日志应用于stateMachine中

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

