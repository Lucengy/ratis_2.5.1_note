## 1. 前言

根据Ratis981中的comment信息，溯源而来

![1741133661379](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1741133661379.png)

这里描述的是FollowerInfo中，其lastRpcdResponseTime是由LeaderStateImpl.addSenders()方法中设置的。

在LeaderStateImpl.addSenders()方法中，使用new FollowerInfoImpl()构造了FollwerInfo对象

```java
Collection<LogAppender> addSenders(Collection<RaftPeer> newPeers, long nextIndex, boolean attendVote， RaftPeerRole role) {
    final Timestamp t = Timestamp.currentTime().addTimeMs(-server.getMaxTimeoutMs());
    final List<LogAppender> newAppenders = newPeers.stream()
        .map(peer -> {
            final FollowerInfo f = new FollowerInfoImpl(server.getMemberId(), peer, t, nextIndex, attendVote);
            peerIdFollowerInfoMap.put(peer.getId(), f);
            if (role == RaftPeerRole.FOLLOWER) {
                raftServerMetrics.addFollower(peer.getId());
                logAppenderMetrics.addFollowerGauges(peer.getId(), f::getNextIndex, f::getMatchIndex, f::getLastRpcTime);
            }
            return server.newLogAppender(this, f);
        }).collect(Collectors.toList());
    senders.addAll(newAppenders);
    return newAppenders;
}
```

而FollowerInfoImpl的构造器中，第三个参数为lastRpcTime。在addSenders()方法中，传入的实参为currentTime

```java
FollowerInfoImpl(RaftGroupMemberId id, RaftPeer peer, Timestamp lastRpcTime, long nextIndex, boolean attendVote) {
    this.name = id + "->" + peer.getId();
    this.infoIndexChange = s -> LOG.info("{}: {}", name, s);
    this.debugIndexChange = s -> LOG.debug("{}: {}", name, s);

    this.peer = peer;
    this.lastRpcResponseTime = new AtomicReference<>(lastRpcTime);
    this.lastRpcSendTime = new AtomicReference<>(lastRpcTime);
    this.lastHeartbeatSendTime = new AtomicReference<>(lastRpcTime);
    this.nextIndex = new RaftLogIndex("nextIndex", nextIndex);
    this.attendVote = attendVote;
  }
```



## 2. FollowerInfo

记录了一个Follower的信息

```java
public interface FollowerInfo {
    String getName();
    RaftPeer getPeer();
    long getMatchIndex();
    //update this follower's matchIndex
    boolean updateMatchIndex(long newMatchIndex);
    
    long updateCommitIndex(long newCommitIndex);
    
    void setSnapshotIndex(long newSnapshotIndex);
    
    void setAttemptedToInstallSnapshot();
    
    boolean hasAttemptedToInstallSnapshot();
    
    long getNextIndex();
    
    void increaseNextIndex(long newNextIndex);
    
    void decreaseNextIndex(long newNextIndex);
    
    void setNextIndex(long newNextIndex);
    
    void updateNextIndex(long newNextIndex);
    
    TimeStamp getLastRpcResponseTime();
    
    void updateLastRpcSendTime();
    
    TimeStamp getLastRpcTime();
    
    TimeStamp getLastHeartbeatSendTime();
}
```

## 3. 实现类FolllowerInfoImpl

```java
class FollowerInfoImpl implements FollowerInfo {
    private final String name;
    private final RaftPeer peer;
    private final AtomicReference<Timestamp> lastRpcSendTime;
    private final AtomicReference<Timestamp> lastRpcResponseTime;
    private final AtomicReference<Timestamp> lastHeartbeatSendTime;
    
    private final RaftLogIndex nextIndex;
    private final RaftLogIndex matchIndex = new RaftLogIndex("matchIndex", 0L);
    private final RfatLogIndex commitIndex = new RaftLogIndex("commitIndex", RaftLog.INVALID_LOG_INDEX);
    private final RaftLogIndex snapshotIndex = new RaftLogIndex("snapshotIndex", 0L);
    
    private volatile boolean attendVote;
    
    private volatile boolean ackInstallSnapshotAttempt = false;
    
    FollowerInfoImpl(RaftGroupMemberId id, RaftPeer peer, Timestamp lastRpcTime, long nextIndex, boolean attendVote) {
    this.name = id + "->" + peer.getId();
    this.infoIndexChange = s -> LOG.info("{}: {}", name, s);
    this.debugIndexChange = s -> LOG.debug("{}: {}", name, s);

    this.peer = peer;
    this.lastRpcResponseTime = new AtomicReference<>(lastRpcTime);
    this.lastRpcSendTime = new AtomicReference<>(lastRpcTime);
    this.lastHeartbeatSendTime = new AtomicReference<>(lastRpcTime);
    this.nextIndex = new RaftLogIndex("nextIndex", nextIndex);
    this.attendVote = attendVote;
  }
}
```

