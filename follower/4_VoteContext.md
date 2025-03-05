## 1. 前言

用来记录Vote过程中的信息

有一个内部的枚举类

```java
enum CheckTermResult {
    FAILED, CHECK_LEADER, SKIP_CHECK_LEADER
}
```



```java
class VoteContext {
    private final RaftServerImpl impl;
    private final RaftConfigurationImpl conf;
    private final Phase phase;
    private final RaftPeerId candidateId;
    
    VoteContext(RaftServerImpl impl, Phase phase, RaftPeerId candidateId) {
        this.impl = impl;
        this.conf = impl.getRaftConf();
        this.phase = phase;
        this.candidateId = candidateId;
    }
    
    private RaftPeer checkConf() {
        if (!conf.containsInConf(candidateId)) {
          reject(candidateId + " is not in current conf " + conf.getCurrentPeers());
          return null;
        }
        return conf.getPeer(candidateId);
  	}
}
```

需要重点关注的几个方法的逻辑

### 1. checkTerm



### 2. checkLeader

可以看到，chekcLeaer中需要检测两方面

1. 该RaftServer是不是leader
2. 该RaftServer为follower时，Leader是不是有效的

```java
private boolean checkLeader() {
    // check if this server is the leader
    final DivisionInfo info = impl.getInfo();
    if (info.isLeader()) {
        if (impl.getRole().getLeaderState().map(LeaderStateImpl::checkLeadership).orElse(false)) {
            return reject("this server is the leader and still has leadership");
        }
    }

    // check if this server is a follower and has a valid leader
    if (info.isFollower()) {
        final RaftPeerId leader = impl.getState().getLeaderId();
        if (leader != null
            && impl.getRole().getFollowerState().map(FollowerState::isCurrentLeaderValid).orElse(false)) {
            return reject("this server is a follower and still has a valid leader " + leader);
        }
    }
    return true;
}
```

首先，看scheckLeadership中的逻辑

```java
  /**
   * See the thesis section 6.2: A leader in Raft steps down
   * if an election timeout elapses without a successful
   * round of heartbeats to a majority of its cluster.
   */

  public boolean checkLeadership() {
      if (!server.getInfo().isLeader()) {
          return false;
      }
      // The initial value of lastRpcResponseTime in FollowerInfo is set by
      // LeaderState::addSenders(), which is fake and used to trigger an
      // immediate round of AppendEntries request. Since candidates collect
      // votes from majority before becoming leader, without seeing higher term,
      // ideally, A leader is legal for election timeout if become leader soon.
      if (server.getRole().getRoleElapsedTimeMs() < server.getMaxTimeoutMs()) {
          return true;
      }

      final List<RaftPeerId> activePeers = senders.stream()
          .filter(sender -> sender.getFollower()
                  .getLastRpcResponseTime()
                  .elapsedTimeMs() <= server.getMaxTimeoutMs())
          .map(LogAppender::getFollowerId)
          .collect(Collectors.toList());

      final RaftConfigurationImpl conf = server.getRaftConf();

      if (conf.hasMajority(activePeers, server.getId())) {
          // leadership check passed
          return true;
      }

      LOG.warn(this + ": Lost leadership on term: " + currentTerm
               + ". Election timeout: " + server.getMaxTimeoutMs() + "ms"
               + ". In charge for: " + server.getRole().getRoleElapsedTimeMs() + "ms"
               + ". Conf: " + conf);
      senders.stream().map(LogAppender::getFollower).forEach(f -> LOG.warn("Follower {}", f));

      // step down as follower
      stepDown(currentTerm, StepDownReason.LOST_MAJORITY_HEARTBEATS);
      return false;
  }
```

