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

返回的是TermIndex，并不是logEntry

1. 首先找raftLog最后一条logEntry的TermIndex，如果raftLog中没有日志
2. 查找最新一次snapshot中的TermIndex

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

### 8. recognizeLeader方法

根据JavaDoc，是对incoming AppendEntries rpc中的leader信息进行校验，如果校验通过，返回true。

这里current是本raft peer，peerLeaderId和leaderTerm是incoming AppendEntries rpc中携带的信息

1. 如果leaderTerm < current，返回false
2. 如果leaderTerm > current 或者 （leaderTerm>= current且当前没有leaderId）时，返回true
3. 如果leaderTerm == current，那么当peerLeaderId==leaderId时，返回true，不等于leaderId时，返回false

```java
/**
   * Check if accept the leader selfId and term from the incoming AppendEntries rpc.
   * If accept, update the current state.
   * @return true if the check passes
 */
boolean recognizeLeader(RaftPeerId peerLeaderId, long leaderTerm) {
    final long current = currentTerm.get();
    if (leaderTerm < current) {
        return false;
    } else if (leaderTerm > current || this.leaderId == null) {
        // If the request indicates a term that is greater than the current term
        // or no leader has been set for the current term, make sure to update
        // leader and term later
        return true;
    }
    return this.leaderId.equals(peerLeaderId);
}
```

### 9. snapshot相关方法

installSnapshot()方法为本机自行installSnapshot，并不是leader向follower/listener发送install rpc

这里的步骤为

1. 暂停stateMachine
2. 调用SnapshotManager.installSnapshot()方法
3. 调用updateInstalledSnapshotIndex()

```java
void installSnapshot(InstallSnapshotRequestProto request) throws IOException {
    // TODO: verify that we need to install the snapshot
    StateMachine sm = server.getStateMachine();
    sm.pause(); // pause the SM to prepare for install snapshot
    snapshotManager.installSnapshot(request, sm, getStorage().getStorageDir());
    updateInstalledSnapshotIndex(TermIndex.valueOf(request.getSnapshotChunk().getTermIndex()));
}
```

updateInstalledSnapshotIndex()方法主要做两件事

1. 通知raftLog，这是告知raftLog purge掉已经takeSnapshot部分的log
2. 更新实例变量latestInstalledSnapshot，这是用来缓存最新snapshot的TermIndex信息的

getLatestSnapshot()方法用来获取最新的snapshot

```java
private SnapshotInfo getLatestSnapshot() {
    return server.getStateMachine().getLatestSnapshot();
}
```

getLatestInstalledSnapshotIndex()，及时获取缓存中最新snapshot的TermIndex信息

```java
long getLatestInstalledSnapshotIndex() {
    final TermIndex ti = latestInstalledSnapshot.get();
    return ti != null? ti.getIndex(): RaftLog.INVALID_LOG_INDEX;
}
```

getSnapshotIndex()用来获取最新snapshot的index信息，这个信息在上述两个方法中寻找最大值

```java
/**
   * The last index included in either the latestSnapshot or the latestInstalledSnapshot
   * @return the last snapshot index
 */
long getSnapshotIndex() {
    final SnapshotInfo s = getLatestSnapshot();
    final long latestSnapshotIndex = s != null ? s.getIndex() : RaftLog.INVALID_LOG_INDEX;
    return Math.max(latestSnapshotIndex, getLatestInstalledSnapshotIndex());
}
```

### 10. getNextIndex()方法

获取下一条logEntry的index信息，这里有点小疑问了，这里是寻找raftLog和snapshot中的最大值。按着正常逻辑来说，应该是先寻找raftLog，如果raftLog是空的，再从最新snapshot中获取TermIndex。

```java
long getNextIndex() {
    final long logNextIndex = getLog().getNextIndex();
    final long snapshotNextIndex = getLog().getSnapshotIndex() + 1;
    return Math.max(logNextIndex, snapshotNextIndex);
}
```

这里看RaftLog.getNextIndex()方法

```java
default long getNextIndex() {
    final TermIndex last = getLastEntryTermIndex();
    if (last == null) {
        // if the log is empty, the last committed index should be consistent with
        // the last index included in the latest snapshot.
        return getLastCommittedIndex() + 1;
    }
    return last.getIndex() + 1;
}
```

这里当last==null时，也就是当raftLog为空时的情况，那么根据这里的JavaDoc以及RaftLogBase中的实现，可以看到，当

```java
@Override
public long getLastCommittedIndex() {
    return commitIndex.get();
}
```

