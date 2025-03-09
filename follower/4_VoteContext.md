## 1. 前言

用来记录Vote过程中的信息

有一个内部的枚举类

```java
enum CheckTermResult {
    FAILED, CHECK_LEADER, SKIP_CHECK_LEADER
}
```

从RequestVote流程过来的

```java
final VoteContext context = new VoteContext(this, phase, candidateId);
```

VoteContext构造前期中的RaftServerImpl，不需要多讲，phase和candidateId都是发起election的candidate的RPC中携带的信息，这里并没有存储term等相关信息哈？？？

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

这里存在三种情况，需要结合recognizedCandidate()方法去理解

* FAILED，两种情况：当candidateTerm < currentTerm时；当candidateTerm==currentTerm，即当该follower在本term中已经投票，且投票不为candidate时
* CHECK_LEADER，两种情况：当该rpc为preVote时；当candidateTerm==currentTerm，即当该follower在本term中未投票或者(已投票且投给candidate时)
* SKIP_CHECK_LEADER：一种情况：当candidateTerm>currentTerm时

### 2. checkLeader

可以看到，chekcLeaer中需要检测两方面。

1. 该RaftServer是不是leader
2. 该RaftServer为follower时，Leader是不是有效的

​	这里有一点拗口的地方，checkLeader返回值是反着来的，即

* 如果存在leader，返回false
* 如果不存在leader，返回ture

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

对于CheckTermResult的返回值，recognizeCandiate()方法的处理逻辑

* 当candidate不在该RaftPeer的conf中，返回null
* 当checkTerm()方法返回FAILED时，返回null
* 当checkTerm()方法返回CHECK_LEADER，但是checkLeader()返回false，即当前存在leader时，返回null。这里的checkLeader()方法的返回值和常规逻辑是反着的，存在leader时，返回false；不存在leader时，返回true
* 其他情况，返回candidate

这里的调用方在decideVote，一般情况下，是将recognizeCandidate的返回值作为实参，传递给decideVote()方法，而decideVote()方法，首先就是对RaftPeer candidate是否null进行判断，当candidate为null时，该RaftPeer拒绝投票

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

相当于在decideVote()方法之前，对于term的检查已经完毕，这一步是用来比较candidate发过来的RPC中携带的logEntry和该RaftServer中的logEntry谁的log更加的update-to-date。这部分的逻辑在ServerState.compareLog()方法中

```java
static int compareLog(TermIndex lastEntry, TermIndex candidateLastEntry) {
    if (lastEntry == null) {
        // If the lastEntry of candidate is null, the proto will transfer an empty TermIndexProto,
        // then term and index of candidateLastEntry will both be 0.
        // Besides, candidateLastEntry comes from proto now, it never be null.
        // But we still check candidateLastEntry == null here,
        // to avoid candidateLastEntry did not come from proto in future.
        if (candidateLastEntry == null ||
            (candidateLastEntry.getTerm() == 0 && candidateLastEntry.getIndex() == 0)) {
            return 0;
        }
        return -1;
    } else if (candidateLastEntry == null) {
        return 1;
    }

    return lastEntry.compareTo(candidateLastEntry);
}
```

