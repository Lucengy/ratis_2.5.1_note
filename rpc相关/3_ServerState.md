## 1. 前言

根据RaftServerImpl构造器中对ServerState的初始化

**this.state = new ServerState(id, group, stateMachine, this, option, properties);**

可以看出，ServerState是跟RaftServerImpl是统一层面的概念。ServerState是每一个RaftServerImpl都会用到的对象，leader专属的类是LeaderState

## 2. ServerState

### 1. 实例变量

```java
  private final RaftGroupMemberId memberId; 
  private final RaftServerImpl server;
  /** Raft log */
  private final MemoizedSupplier<RaftLog> log;
  /** Raft configuration */
  private final ConfigurationManager configurationManager;
  /** The thread that applies committed log entries to the state machine */
  private final MemoizedSupplier<StateMachineUpdater> stateMachineUpdater;
  /** local storage for log and snapshot */
  private final MemoizedCheckedSupplier<RaftStorageImpl, IOException> raftStorage;
  private final SnapshotManager snapshotManager;
  private volatile Timestamp lastNoLeaderTime;
  private final TimeDuration noLeaderTimeout;

  private final ReadRequests readRequests;

  /**
   * Latest term server has seen.
   * Initialized to 0 on first boot, increases monotonically.
   */
  private final AtomicLong currentTerm = new AtomicLong();
  /**
   * The server ID of the leader for this term. Null means either there is
   * no leader for this term yet or this server does not know who it is yet.
   */
  private volatile RaftPeerId leaderId;
  /**
   * Candidate that this peer granted vote for in current term (or null if none)
   */
  private volatile RaftPeerId votedFor;

  /**
   * Latest installed snapshot for this server. This maybe different than StateMachine's latest
   * snapshot. Once we successfully install a snapshot, the SM may not pick it up immediately.
   * Further, this will not get updated when SM does snapshots itself.
   */
  private final AtomicReference<TermIndex> latestInstalledSnapshot = new AtomicReference<>();
```

ServerState的入口方法为start()方法，该方法用来启动StateMachineUpdater这个线程

```java
void start() {
    stateMachineUpdater.get().start();
}
```

### 2. initialize方法

初始化，这里包含

* 存储方面的初始化，即RaftStorage
* 配置方面的初始化，即RaftConfiguration
* 状态机的初始化，即StateMachine
* load元数据到内存，即term和votedFor信息

```java
void initialize(StateMachine stateMachine) throws IOException {
    // initialize raft storage
    final RaftStorageImpl storage = raftStorage.get();
    // read configuration from the storage
    Optional.ofNullable(storage.readRaftConfiguration()).ifPresent(this::setRaftConf);

    stateMachine.initialize(server.getRaftServer(), getMemberId().getGroupId(), storage);

    // we cannot apply log entries to the state machine in this step, since we
    // do not know whether the local log entries have been committed.
    final RaftStorageMetadata metadata = log.get().loadMetadata();
    currentTerm.set(metadata.getTerm());
    votedFor = metadata.getVotedFor();
}
```

### 3. initRaftLog方法

初始化raftLog，在构造器中调用

```java
private RaftLog initRaftLog(LongSupplier getSnapshotIndexFromStateMachine, RaftProperties prop) {
    try {
        return initRaftLog(getMemberId(), server, getStorage(), this::setRaftConf,
                           getSnapshotIndexFromStateMachine, prop);
    } catch (IOException e) {
        throw new IllegalStateException(getMemberId() + ": Failed to initRaftLog.", e);
    }
}

private static RaftLog initRaftLog(RaftGroupMemberId memberId, RaftServerImpl server, RaftStorage storage,
                                   Consumer<LogEntryProto> logConsumer, LongSupplier getSnapshotIndexFromStateMachine,
                                   RaftProperties prop) throws IOException {
    final RaftLog log;
    if (RaftServerConfigKeys.Log.useMemory(prop)) {
        log = new MemoryRaftLog(memberId, getSnapshotIndexFromStateMachine, prop);
    } else {
        log = new SegmentedRaftLog(memberId, server,
                                   server.getStateMachine(),
                                   server::notifyTruncatedLogEntry,
                                   server::submitUpdateCommitEvent,
                                   storage, getSnapshotIndexFromStateMachine, prop);
    }
    log.open(log.getSnapshotIndex(), logConsumer);
    return log;
}
```

### 4. setLeader方法

这里有做了三件事

1. stateMachine.event().notifyLeaderChanged()
2. RaftServerImpl.finishTransferLeadership()
3. RaftServerImpl.onGroupLeaderElected()

```java
void setLeader(RaftPeerId newLeaderId, Object op) {
    if (!Objects.equals(leaderId, newLeaderId)) {
        String suffix;
        if (newLeaderId == null) {
            // reset the time stamp when a null leader is assigned
            lastNoLeaderTime = Timestamp.currentTime();
            suffix = "";
        } else {
            Timestamp previous = lastNoLeaderTime;
            lastNoLeaderTime = null;
            suffix = ", leader elected after " + previous.elapsedTimeMs() + "ms";
            server.getStateMachine().event().notifyLeaderChanged(getMemberId(), newLeaderId);
        }
        LOG.info("{}: change Leader from {} to {} at term {} for {}{}",
                 getMemberId(), leaderId, newLeaderId, getCurrentTerm(), op, suffix);
        leaderId = newLeaderId;
        if (leaderId != null) {
            server.finishTransferLeadership();
            server.onGroupLeaderElected();
        }
    }
}
```

### 5. updateCurrentTerm方法

```java
boolean updateCurrentTerm(long newTerm) {
    final long current = currentTerm.getAndUpdate(curTerm -> Math.max(curTerm, newTerm));
    if (newTerm > current) {
        votedFor = null;
        setLeader(null, "updateCurrentTerm");
        return true;
    }
    return false;
}
```

### 6. initElection方法

根据JavaDoc，这是当前raft peer从follower状态转换为candidate状态，存在两种情况

1. 当发起的是preVote请求时，只是返回当前的term
2. 当发起的是election请求时，将当前term+1，将votedFor修改为自己，即raft peer在发起election时，默认都是给自己投票的，以及将对应term和votedFor元数据信息写到磁盘上

```java
/**
   * Become a candidate and start leader election
 */
LeaderElection.ConfAndTerm initElection(Phase phase) throws IOException {
    setLeader(null, phase);
    final long term;
    if (phase == Phase.PRE_VOTE) {
        term = getCurrentTerm();
    } else if (phase == Phase.ELECTION) {
        term = currentTerm.incrementAndGet();
        votedFor = getMemberId().getPeerId();
        persistMetadata();
    } else {
        throw new IllegalArgumentException("Unexpected phase " + phase);
    }
    return new LeaderElection.ConfAndTerm(getRaftConf(), term);
}
```

### 7. getLastEntry方法

```java
TermIndex getLastEntry() {
    TermIndex lastEntry = getLog().getLastEntryTermIndex();
    if (lastEntry == null) {
        // lastEntry may need to be derived from snapshot
        SnapshotInfo snapshot = getLatestSnapshot();
        if (snapshot != null) {
            lastEntry = snapshot.getTermIndex();
        }
    }

    return lastEntry;
}

RaftLog getLog() {
    if (!log.isInitialized()) {
        throw new IllegalStateException(getMemberId() + ": log is uninitialized.");
    }
    return log.get();
}
```

