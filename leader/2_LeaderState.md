## 0. 前言

方法的入口在RaftServerImpl.changeToLeader() --> RoleInfo.startLeaderState(RaftServerImpl) --> LeaderStateImpl.start()

根据RATIS-353，leader在掌权时，应该发送一个placeHolder的logEntry，之前发送的应该是no-op，即心跳。现在发送的是raftConf。这里注意placeHolder的logIndex使用的是raftLog.getNextIndex()，也对哈，意味着每个leader掌权时都会有一条conf的logEntry，然后在本地raftLog中添加了一条logEntry，调用EventProcessor.start()Daemon线程监测和更新leaderStateImpl的状态信息。senders保存着到各个Follower的Rpc类，调用其start()方法

```java
  LogEntryProto start() {
    // In the beginning of the new term, replicate a conf entry in order
    // to finally commit entries in the previous term.
    // Also this message can help identify the last committed index and the conf.
    final LogEntryProto placeHolder = LogProtoUtils.toLogEntryProto(
        server.getRaftConf(), server.getState().getCurrentTerm(), raftLog.getNextIndex());
    CodeInjectionForTesting.execute(APPEND_PLACEHOLDER,
        server.getId().toString(), null);
    raftLog.append(Collections.singletonList(placeHolder));
    processor.start();
    senders.forEach(LogAppender::start);
    return placeHolder;
  }
```



## 1. 接口

需要学习的是其接口定义，以**on**开头的方法都是事件驱动型的方法，当收到对应事件时，会被调用

```java
public interface LeaderState {
  /** The reasons that this leader steps down and becomes a follower. */
  enum StepDownReason {
    HIGHER_TERM, HIGHER_PRIORITY, LOST_MAJORITY_HEARTBEATS, STATE_MACHINE_EXCEPTION, JVM_PAUSE, FORCE;

    private final String longName = JavaUtils.getClassSimpleName(getClass()) + ":" + name();

    @Override
    public String toString() {
      return longName;
    }
  }

  /** Restart the given {@link LogAppender}. */
  void restart(LogAppender appender);

  /** @return a new {@link AppendEntriesRequestProto} object. */
  AppendEntriesRequestProto newAppendEntriesRequestProto(FollowerInfo follower,
      List<LogEntryProto> entries, TermIndex previous, long callId);

  /** Check if the follower is healthy. */
  void checkHealth(FollowerInfo follower);

  /** Handle the event that the follower has replied a term. */
  boolean onFollowerTerm(FollowerInfo follower, long followerTerm);

  /** Handle the event that the follower has replied a commit index. */
  void onFollowerCommitIndex(FollowerInfo follower, long commitIndex);

  /** Handle the event that the follower has replied a success append entries. */
  void onFollowerSuccessAppendEntries(FollowerInfo follower);

  /** Check if a follower is bootstrapping. */
  boolean isFollowerBootstrapping(FollowerInfo follower);

  /** Received an {@link AppendEntriesReplyProto} */
  void onAppendEntriesReply(FollowerInfo follower, AppendEntriesReplyProto reply);

}
```

## 2. LeaderStateImpl实现

整体比较复杂，其内部类较多。根据JavaDoc描述，这个类包含了三种不同的线程

1. RPC senders: 每个线程负责向一个follower发送appendLog
2. EventProcessor: a single thread，根据rpc结果更新leader的状态信息
3. PendingRequestHandler: 当logEntry成功commit后，使用该线程将结果发送给client

```
Sates for leader only. It contains three different types of processors:
 1. RPC senders: each thread is appending log to a follower
 2. EventProcessor: a single thread updating the raft server's state based on status of log appending response
 3. PendingRequestHandler: a handler sending back responses to clients when corresponding log entries are committed.
```

### 1. 内部类BootStrapProgress

枚举类

```java
private enum BootStrapProgress {
    NOPROGRESS< PROGRESSING, CAUGHTUP
}
```

### 2. StateUpdateEvent

```java
static class StateUpdateEvent {
    private enum Type {
      STEP_DOWN, UPDATE_COMMIT, CHECK_STAGING
    }
    
    private final Type type;
    private final long newTerm;
    private final Runnable handler;
    
    StateUpdateEvent(Type type, long newTerm, Runnable handler) {
        this.type = type;
        this.newTerm = newTerm;
        this.handler = handler;
    }
    
    void execute() {
        handler.run();
    }
}
```

### 3. 内部类EventQueue

```java
private class EventQueue {
    private final String name = server.getMemberId() + "-" + JavaUtils.getClassSimpleName(getClass());
    private final BlockingQueue<StateUpdateEvent> queue = new ArrayBlockingQueue<>(4096);
    
    void submit(StateUpdateEvent event) {
        try {
            queue.put(event);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    StateUpdateEvent poll() {
        final StateUpdateEvent e;
        
        try {
            e = queue.poll(server.getMaxTimeoutMs(), TimeUnit.MILLISECONS);
        } catch (InterruptedException ie) {
            Thread.currentThread().interrupt();
            String s = this + ": poll() is interrupted";
            if(!running) {
                LOG.info(s + " gracefully");
                return null;
            } else {
                throw new IllegalStateException(s + " UNEXPECTEDLY", ie);
            }
        }
        
        if(e != null) {
            while(e.equals(queue.peek())) {
                queue.poll();
            }
        }
        
        return e;
    }
}
```

### 4. 内部类SenderList

持有一个LogAppender的list

```java
static class SenderList {
    private final List<LogAppender> senders;
}
```

### 5. 内部类MinMajorityMax

```java
static class MinMajorityMax {
    private final long min;
    private final long majority;
    private final long max;
    
    MinMajorityMax(long min, long majority, long max) {
        this.min = min;
        this.majority = majority;
        this.max = max;
    }
    
    MinMajority combine(MinMajorityMax max) {
        return ew MinMajorityMax(
        Math.min(this.min, that.min),
        Math.min(this.majority, that.majority),
        Math.min(this.max, that.max)
        );
    }
    
    static MinMajorityMax valueOf(long[] sorted) {
        return new MinMajorityMax(sorted[0], getMajority(sorted), getMax(sorted));
    }
    
    static MinMajorityMax valueOf(long[] sorted, long gapThreshold) {
        long majority = getMajority(sorted);
        long min = sorted[0];
        
        if(gapThreshold != -1 && (majority - min) > gapThreshold) {
            majority = min;
        }
        
        return new MinMajorityMax(min, majority, getMax(sorted));
    }
    
    static long getMajority(long[] sorted) {
        return sorted[(sorted.length - 1) / 2];
    }
    
    static long getMax(long[] sorted) {
        return sorted[sorted.length - 1];
    }
}
```

### 6. 内部类EventProcessor

根据JavaDoc

```java
/**
   * The processor thread takes the responsibility to update the raft server's
   * state, such as changing to follower, or updating the committed index.
*/
```



```java
private class EventProcessor extends Daemon {
    public EventProcessor(String name) {
        setName(name);
    }
    @Override
    public void run() {
        // apply an empty message; check if necessary to replicate (new) conf
        prepare();

        while (running) {
            final StateUpdateEvent event = eventQueue.poll();
            synchronized(server) {
                if (running) {
                    if (event != null) {
                        event.execute();
                    } else if (inStagingState()) {
                        checkStaging();
                    } else {
                        yieldLeaderToHigherPriorityPeer();
                        checkLeadership();
                    }
                }
            }
        }
    }
}
```



### 实例变量

这里很多实例变量都是使用final进行修饰的，证明在一轮Term中，LeaderStateImpl中很多实例变量都不会被修改

```java
private final StateUpdateEvent updateCommitEvent = 
    new StateUpdateEvent(StateUpdateEvent.Type.UPDATE_COMMIT, -1, this::updateCommit);
private final StateUpdateEvent checkingStagingEvent =
    new StateUpdateEvent(StateUpdateEvent.Type.CHECK_STATING, -1, this::checkStaing);

private final String name;
private final RaftServerImpl server;
private final RaftLog raftLog;
private final long currentTerm;
private volatile ConfigurationStaingState stagingState;
private List<List<RaftPeerId>> voterLists;
private final Map<RaftPeerId, FollowerInfo> peerIdFollowerInfoMap = new ConcurrentHashMap<>();

private final SenderList senders;

private final EventQueue eventQueue;
private final EventProcessor processor;
private final PendingRequests pendingRequests;
private final WatchReqeusts watchRequests;
private final MessageStreamRequests messageStreamRequests;

private volatile boolean running = true;
private final int stagingCatchupGap;
private final long placeHolderIndex;

private final long followerMaxGapThreshold;
private final PendingStepDown pendingStepDown;
```

这里的voterLists方法也比较疑惑，是一个二维列表，其在构造器中被赋值

```java
voterLists = divideFollowers(conf);
```

接下来看divideFollowers()方法，这里的list可能有一个元素，也可能有两个元素，取决于当前leader的状态。如果处在configurationChange阶段，那么list存在第二个元素，且第二个元素为oldConf中的raftPeer集合。第一个元素不变，都为currentConf中的raftPeer集合

```java
private List<List<RaftPeerId>> divideFollowers(RaftConfigurationImpl conf) {
    List<List<RaftPeerId>> lists = new ArrayList<>(2);
    List<RaftPeerId> listForNew = senders.stream()
        .map(LogAppender::getFollowerId)
        .filter(conf::containsInConf)
        .collect(Collectors.toList());
    lists.add(listForNew);
    if (conf.isTransitional()) {
        List<RaftPeerId> listForOld = senders.stream()
            .map(LogAppender::getFollowerId)
            .filter(conf::containsInOldConf)
            .collect(Collectors.toList());
        lists.add(listForOld);
    }
    return lists;
}
```



EventQueue的submit()方法的调用方，这里调用方为submitUpdateCommitEvent方法，该方法提交的为updateCommitEvent对象，该对象为LeaderStateImpl的实例变量，默认是赋值的，其nahdler调用的为this.updateCommit方法，那么接下来看updateCommit方法

```JAVA
void submitUpdateCommitEvent() {
    eventQueue.submit(updateCommitEvent);
}
```



```java
private void updateCommit() {
    getMajorityMin(FollowerInfo::getMatchIndex, raftLog::getFlushIndex,
                   followerMaxGapThreshold)
        .ifPresent(m -> updateCommit(m.majority, m.min));
}
```

这里调用了getMajorityMin方法和updateCommit(long, long)方法，首先看getMajorityMin()方法。这里有四个return语句，注意有返回null的情况，即Optional.empty()

* 当followers为空，且不包含本身的情况，返回null
* 当conf处于transitional状态，oldConf为且不包含自身的情况，返回null

```java
private Optional<MinMajorityMax> getMajorityMin(ToLongFunction<FollowerInfo> followerIndex,
      LongSupplier logIndex, long gapThreshold) {
    final RaftPeerId selfId = server.getId();
    final RaftConfigurationImpl conf = server.getRaftConf();

    final List<RaftPeerId> followers = voterLists.get(0);
    final boolean includeSelf = conf.containsInConf(selfId);
    if (followers.isEmpty() && !includeSelf) {
        return Optional.empty();
    }

    final long[] indicesInNewConf = getSorted(followers, includeSelf, followerIndex, logIndex);
    final MinMajorityMax newConf = MinMajorityMax.valueOf(indicesInNewConf, gapThreshold);

    if (!conf.isTransitional()) {
        return Optional.of(newConf);
    } else { // configuration is in transitional state
        final List<RaftPeerId> oldFollowers = voterLists.get(1);
        final boolean includeSelfInOldConf = conf.containsInOldConf(selfId);
        if (oldFollowers.isEmpty() && !includeSelfInOldConf) {
            return Optional.empty();
        }

        final long[] indicesInOldConf = getSorted(oldFollowers, includeSelfInOldConf, followerIndex, logIndex);
        final MinMajorityMax oldConf = MinMajorityMax.valueOf(indicesInOldConf, followerMaxGapThreshold);
        return Optional.of(newConf.combine(oldConf));
    }
}
```

这里用到了getSorted()方法，getSorted()方法是对followers的logIndex进行排序，使用形参中的ToLongFunction\<FollwerInfo>方法，获取logIndex值；同时，根据includeSelf参数决定是否将自己的logIndex也参与到排序返回中

```java
private long[] getSorted(List<RaftPeerId> followerIDs, boolean includeSelf,
      ToLongFunction<FollowerInfo> getFollowerIndex, LongSupplier getLogIndex) {
    final int length = includeSelf ? followerIDs.size() + 1 : followerIDs.size();
    if (length == 0) {
        throw new IllegalArgumentException("followers.size() == "
          + followerIDs.size() + " and includeSelf == " + includeSelf);
    }

    final long[] indices = new long[length];
    List<FollowerInfo> followerInfos = getFollowerInfos(followerIDs);
    for (int i = 0; i < followerInfos.size(); i++) {
        indices[i] = getFollowerIndex.applyAsLong(followerInfos.get(i));
    }

    if (includeSelf) {
        // note that we also need to wait for the local disk I/O
        indices[length - 1] = getLogIndex.getAsLong();
    }

    Arrays.sort(indices);
    return indices;
}
```

接下来看updateCommit(long, long)方法，这里在调用ServerState.updateCommitIndex()方法更新内存中的状态信息后，调用updateCommit(LogEntryHeader[])方法。该方法只是用来确定待commit的logEntries中是否有configuration相关的logEntry，如果存在，调用checkAndUpdateConfiguration()方法

```java
private void updateCommit(long majority, long min) {
    final long oldLastCommitted = raftLog.getLastCommittedIndex();
    if (majority > oldLastCommitted) {
        // Get the headers before updating commit index since the log can be purged after a snapshot
        final LogEntryHeader[] entriesToCommit = raftLog.getEntries(oldLastCommitted + 1, majority + 1);

        if (server.getState().updateCommitIndex(majority, currentTerm, true)) {
            updateCommit(entriesToCommit);
        }
    }
    watchRequests.update(ReplicationLevel.ALL, min);
}
```

这里调用了ServerState.updateCommitIndex()方法

```java
boolean updateCommitIndex(long majorityIndex, long curTerm, boolean isLeader) {
    if (getLog().updateCommitIndex(majorityIndex, curTerm, isLeader)) {
        getStateMachineUpdater().notifyUpdater();
        return true;
    }
    return false;
}
```

ServerState.updateCommitIndex()方法调用了RaftLog.updateCommitIndex(long, long)方法，同时将该事件通知给stateMachineUpdater对象。RaftLog.updateCommitIndex()方法更新内存中commitIndex的信息

```java
@Override
  public boolean updateCommitIndex(long majorityIndex, long currentTerm, boolean isLeader) {
      try(AutoCloseableLock writeLock = writeLock()) {
          final long oldCommittedIndex = getLastCommittedIndex();
          final long newCommitIndex = Math.min(majorityIndex, getFlushIndex());
          if (oldCommittedIndex < newCommitIndex) {
              if (!isLeader) {
                  commitIndex.updateIncreasingly(newCommitIndex, traceIndexChange);
                  return true;
              }

              // Only update last committed index for current term. See §5.4.2 in paper for details.
              final TermIndex entry = getTermIndex(newCommitIndex);
              if (entry != null && entry.getTerm() == currentTerm) {
                  commitIndex.updateIncreasingly(newCommitIndex, traceIndexChange);
                  return true;
              }
          }
      }
      return false;
  }
```









 







