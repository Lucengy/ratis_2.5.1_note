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

```java
private CheckTermResult checkTerm(long candidateTerm) {
    if (phase == Phase.PRE_VOTE) {
        return CheckTermResult.CHECK_LEADER;
    }
    // check term
    final ServerState state = impl.getState();
    final long currentTerm = state.getCurrentTerm();
    if (currentTerm > candidateTerm) {
        reject("current term " + currentTerm + " > candidate's term " + candidateTerm);
        return CheckTermResult.FAILED;
    } else if (currentTerm == candidateTerm) {
        // check if this server has already voted
        final RaftPeerId votedFor = state.getVotedFor();
        if (votedFor != null && !votedFor.equals(candidateId)) {
            reject("already has voted for " + votedFor + " at current term " + currentTerm);
            return CheckTermResult.FAILED;
        }
        return CheckTermResult.CHECK_LEADER;
    } else {
        return CheckTermResult.SKIP_CHECK_LEADER; //currentTerm < candidateTerm
    }
}
```



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

首先，看checkLeadership中的逻辑。这里先简单聊一下，详细的解释放到LeaderStateImpl中去聊。这里的主逻辑无非是

```
每一个LogAppender对应一个Follower，根据其lastResponseTime的持续时间判断是否小于leader的超时时间。如果是，证明leader依旧掌权，如果不是，应该是发生了脑裂，那么此时leader应该下线退为follower.
```

这里有一种意外情况，就是当leader刚刚掌权时，并不符合上述的要求，所以有了前置的判断，判断leader是否刚刚掌权，如果是，直接返回true

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

### 3. recognizeCandidate

```java
RaftPeer recognizeCandidate(long candidateTerm) {
    final RaftPeer candidate = checkConf();
    if(candidate == null)
        return null;
    
    final CheckTermResult r = checkTerm(candidateTerm);
    if(r == CheckTermResult.FAILED) {
        return null;
    }
    
    if(r == CheckTermResult.CHECK_LEADER && !checkLeader())
        return null;
    
    return candiadate;
}
```

这里用到了checkConf方法，根据JavaDoc，该方法用来检查指定的candidate是否在当前的conf中，VoteContext对应的是一个节点的投票信息，持有对应节点的RaftPeerId，即candiateId，同时其实例变量均使用final进行修饰，所以checkConf()方法并没有形参

```java
private RaftPeer checkConf() {
    if(!conf.containsInConf(candidateId)) {
        reject(candidateId + " is not in current conf " + conf.getCurrentPeers());
        return null;
    }
    
    return conf.getPeer(candidateId);
}
```

### 4. decideVote

```java
boolean decideVote(RaftPeer candidate, TermIndex candidateLastEntry) {
    if (candidate == null) {
        return false;
    }
    // Check last log entry
    final TermIndex lastEntry = impl.getState().getLastEntry();
    final int compare = ServerState.compareLog(lastEntry, candidateLastEntry);
    if (compare < 0) {
        return log(true, "our last entry " + lastEntry + " < candidate's last entry " + candidateLastEntry);
    } else if (compare > 0) {
        return reject("our last entry " + lastEntry + " > candidate's last entry " + candidateLastEntry);
    }

    // Check priority
    final RaftPeer peer = conf.getPeer(impl.getId());
    if (peer == null) {
        return reject("our server " + impl.getId() + " is not in the conf " + conf);
    }
    final int priority = peer.getPriority();
    if (priority <= candidate.getPriority()) {
        return log(true, "our priority " + priority + " <= candidate's priority " + candidate.getPriority());
    } else {
        return reject("our priority " + priority + " > candidate's priority " + candidate.getPriority());
    }
}
```

