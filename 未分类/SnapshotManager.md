## 1.  前言

根据；Raft(expand)论文描述

```
Raft also includes a small amount of metadata in th snapshot: the last included index is the index of the last entry in the log that the snapshot replaces(the last entry the state machine has applied), and the last included term is the term of this entry. 
These are preserved to support the AppendEntries consistency check for the first log entry following the snapshot, since that entry needs a previous log index and term.
```

还有一点元数据需要存储，这里分开记录

```
To enable cluster membership changes, the snapshot also includes the latest configuration in the log as of last included index.
```

接下来看SnapshotInfo的接口，接口中只是定义了TermIndex，即snapshot中包含的最后一条applied的log entry的term和index，关于configuration的部分并没有在SnapshotInfo接口中进行定义

```java
public interface SnapshotInfo {
    TermIndex getTermIndex();
    
    default long getTerm() {
        return getTermIndex().getTerm();
    }
    
    default long getIndex() {
        return getTermIndex().getIndex();
    }
    
    List<FileInfo> getFiles();
}
```

这里有两个问题需要抛出来

1. leader和follower触发takeSnapshot以及shouldTakeSnapshot的动作是否一致
2. takeSnapshot的入口方法在哪里

有关第一个问题，是一致的，这里要等到回答完第二个问题后，再来看

有关第一个问题，其入口方法在StateMachineUpdater中，如Ratis-1473中所描述：

![1738544902589](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1738544902589.png)

该Jira是为了给Admin提供了一个takeSnapshot的接口，但同时也为我们解释了由StateMachineUpdater这个单独的线程来触发takeSnapshot动作。

这里还有一点需要记录的，根据RATIS-338，leader在RPC中加入了commitIndexInfo，即各个peer的commitIndex信息，分别在appendEntries和ClientReply中加入了这部分信息。在takeSnapshot动作中，raftLog需要purge掉过期的logEntries，那么从哪里开始purge呢？在RATIS-850中，最初的设计是从选取所有peer中commitIndex以及snapshotIndex的最小值作为purge的index，在Ozone中发现如果存在掉队的follower，那么其commitIndex会较小，这样其他peer就会停止purge日志，jira中是这样描述的

```
Ratis logs are purged only up to the least commit index on all the peers. But if one peer is down, it stop log purging on all the peers. 
```

因此，我们可以选取snapshotInfo的index作为log purge的index，这样做可以及时的清除purge log，但也会有一点问题，当leader已经purge掉对应的log entries后，向掉队的follower只能发送snapshotInfo。

```protobuf
message CommitInfoProto {
  RaftPeerProto server = 1;
  uint64 commitIndex = 2;
}

message AppendEntriesRequestProto {
  RaftRpcRequestProto serverRequest = 1;
  uint64 leaderTerm = 2;
  TermIndexProto previousLog = 3;//consistency check, When sending an AppendEntries RPC,the leader
  //includes the index and term of the entry in its log that immediately precedes the new entries.
  repeated LogEntryProto entries = 4;
  uint64 leaderCommit = 5;//the leader keeps track of the highest index it knows to be committed,
  //and it includes that index in future AppendEntries RPCs(including heartbeats) so that the other
  //servers eventually find out.
  bool initializing = 6;

  repeated CommitInfoProto commitInfos = 15;
}

message RaftClientReplyProto {
  RaftRpcReplyProto rpcReply = 1;
  ClientMessageEntryProto message = 2;

  oneof ExceptionDetails {
    NotLeaderExceptionProto notLeaderException = 3;
    NotReplicatedExceptionProto notReplicatedException = 4;
    StateMachineExceptionProto stateMachineException = 5;
    LeaderNotReadyExceptionProto leaderNotReadyException = 6;
    AlreadyClosedExceptionProto alreadyClosedException = 7;
    ThrowableProto dataStreamException = 8;
    ThrowableProto leaderSteppingDownException = 9;
    ThrowableProto transferLeadershipException = 10;
  }

  uint64 logIndex = 14; // When the request is a write request and the reply is success, the log index of the transaction
  repeated CommitInfoProto commitInfos = 15;
}
```

## 2. StateMachineUpdater

这里存在一个线程对象**Thread updater**，由于StateMachineUpdater继承自Runnable，可想而知，构造器中会对updater进行赋值，将StateMachineUpdater本身包装为线程对象，赋值给updater，其逻辑入口就成为了run()方法，在Ratis中，其线程对象大多都使用该思路

```java
//构造器
StateMachineUpdater(...) {
    ...;
    updater = new Daemon(this);
    ...;
}

//入口方法
void start(){
    updater.start(); //将入口方法改为this.run()方法
}
```

接下来将目光转移到run()方法中

```java
  @Override
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

简单来讲核心就三点

1. waitForCommit(): 等待有新的log entries成为committed的状态
2. 调用applyLog()方法将committed状态的log entries apply到stateMachine中
3.  checkAndTakeSnasphot(): 判断是否需要触发takeSnapshot

其中，waitForCommit()是一个阻塞方法，该方法会阻塞当前线程，直到有新的log entries达到committed状态

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

这里有两个概念需要讲以下，分别为getLastAppliedIndex()和getStateMachineLastAppliedIndex()

```java
  private long getLastAppliedIndex() {
    return appliedIndex.get();
  }

  long getStateMachineLastAppliedIndex() {
    return stateMachine.getLastAppliedTermIndex().getIndex();
  }
```

根据RATIS-614描述，stateMachineLastAppliedTermIndex表示transaction已经完成时的索引，而lastAppliedIndex()表示的是stateMachineUpdater线程调用applyTransaction时的索引

```
Currently Raft leader uses the StateMachineUpdater's lastAppliedIndex to determine if leader is ready to take requests. It should rather use StateMachine's lastAppliedTermIndex because it denotes the index till which the transactions have already been completed whereas StateMachineUpdater's lastAppliedIndex denotes the index till which the applyTransaction call has been made to the StateMachine.
```

只有在shouldTakeSnapshot()方法中，需要判断是否要触发takeSnapshot时判断的才是stateMachineLastAppliedIndex，其余时间判断的都为StateMachineUpdater中的lastAppliedIndex，也是合理，只有当已经完成的transaction的index和当前snapshot的index差值大于threshold时，才会触发takeSnapshot，被减数不应该为已经提交的transaction的index

```java
  private boolean shouldTakeSnapshot() {
    if (state == State.RUNNING && server.getSnapshotRequestHandler().shouldTriggerTakingSnapshot()) {
      return true;
    }
    if (autoSnapshotThreshold == null) {
      return false;
    } else if (shouldStop()) {
      return getLastAppliedIndex() - snapshotIndex.get() > 0;
    }
    return state == State.RUNNING &&
        getStateMachineLastAppliedIndex() - snapshotIndex.get() >= autoSnapshotThreshold;
  }
```

在applyLog()方法中，使用的也是已经提交的transacttion的Index: 在for循环中判断lastAppliedIndex是否大于committed状态的log entries的index，在循环体中更新lastAppliedIndex

```java
private MemoizedSupplier<List<CompletableFuture<Message>>> applyLog() throws RaftLogIOException {
    final MemoizedSupplier<List<CompletableFuture<Message>>> futures = MemoizedSupplier.valueOf(ArrayList::new);
    final long committed = raftLog.getLastCommittedIndex();
    for(long applied; (applied = getLastAppliedIndex()) < committed && state == State.RUNNING && !shouldStop(); ) { //使用的是lastAppliedIndex
      final long nextIndex = applied + 1;
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
        //更新的也是lastAppliedIndex
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

最后是checkAndTakeSnapshot()方法，这里主要逻辑在takeSnapshot()方法中

```java
  private void takeSnapshot() {
    final long i;
    try {
      Timer.Context takeSnapshotTimerContext = stateMachineMetrics.getTakeSnapshotTimer().time();
      i = stateMachine.takeSnapshot();
      takeSnapshotTimerContext.stop();
      server.getSnapshotRequestHandler().completeTakingSnapshot(i);

      final long lastAppliedIndex = getLastAppliedIndex();
      if (i > lastAppliedIndex) {
        throw new StateMachineException(
            "Bug in StateMachine: snapshot index = " + i + " > appliedIndex = " + lastAppliedIndex
            + "; StateMachine class=" +  stateMachine.getClass().getName() + ", stateMachine=" + stateMachine);
      }
      stateMachine.getStateMachineStorage().cleanupOldSnapshots(snapshotRetentionPolicy);
    } catch (IOException e) {
      LOG.error(name + ": Failed to take snapshot", e);
      return;
    }

    if (i >= 0) {
      LOG.info("{}: Took a snapshot at index {}", name, i);
      snapshotIndex.updateIncreasingly(i, infoIndexChange);

      final long purgeIndex;
      if (purgeUptoSnapshotIndex) {
        // We can purge up to snapshot index even if all the peers do not have
        // commitIndex up to this snapshot index.
        purgeIndex = i;
      } else {
        final LongStream commitIndexStream = server.getCommitInfos().stream().mapToLong(
            CommitInfoProto::getCommitIndex);
        purgeIndex = LongStream.concat(LongStream.of(i), commitIndexStream).min().orElse(i);
      }
      raftLog.purge(purgeIndex);
    }
  }
```

在takeSnapshot的最后，需要调用raftLog.purge()方法，清除多余的log entires，purgeIndex的相关内容在前言中已经详细说明，这里不再赘述

## 3. SnapshotManager