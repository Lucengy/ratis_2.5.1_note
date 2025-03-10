## 1. 前言

RoleInfo中存在三个状态相关的类，分别是

- LeaderStateImpl
- FollowerState
- LeaderElection

FollowerState是用来changeToCandidate使用的，而LeaderElection是用来changeToLeader使用的

## 2. 内部类

### 1.  Phase

用来标注RequestVote的两种情况，分别为

* Pre-Vote
* ELECTION

```java
enum Phase {
    PRE_VOTE,
    ELECTION
}
```

### 2. Result

用来标注RequestVote的ack状态

```java
enum Result {
    PASSED,
    REJECT,
    TIMEOUT,
    DISCOVER_A_NEW_TERM,
    SHUTDOWN,
    NOT_IN_CONF
}
```

### 3. ResultAndTerm

顾名思义，封装了Result和Term两个对象

```java
private static class ResultAndTerm {
    private final Result result;
    private final Long term;
    
    public ResultAndTerm(Result result, Long term) {
        this.result = result;
        this.term = term;
    }
    
    long maxTerm(long thatTerm) {
        return this.term != null && this.term > thatTerm ? this.term: thatTerm;
    }
    
    Result getResult() {
        return result;
    }
}
```

### 4. ConfAndTerm

顾名思义，封装了一个RaftConfigurationImpl对象和一个term对象

## 3. LeaderElection

LeaderElection实现了Runnable接口，持有一个Deamon对象，那么在初始化中

```java
this.daemon = new Daemon(this);
```

那么入口方法则为run()方法，run()方法并没有使用循环，煦暖提在askForVotes()方法中，使用死循环一致投票，这里只会对结果返回true/false

```java
  @Override
  public void run() {
    if (!lifeCycle.compareAndTransition(STARTING, RUNNING)) {
      final LifeCycle.State state = lifeCycle.getCurrentState();
      LOG.info("{}: skip running since this is already {}", this, state);
      return;
    }

    final Timer.Context electionContext = server.getLeaderElectionMetrics().getLeaderElectionTimer().time();
    try {
      if (skipPreVote || askForVotes(Phase.PRE_VOTE)) {
        if (askForVotes(Phase.ELECTION)) {
          server.changeToLeader();
        }
      }
    } catch(Exception e) {
      final LifeCycle.State state = lifeCycle.getCurrentState();
      if (state.isClosingOrClosed()) {
        LOG.info("{}: {} is safely ignored since this is already {}",
            this, JavaUtils.getClassSimpleName(e.getClass()), state, e);
      } else {
        if (!server.getInfo().isAlive()) {
          LOG.info("{}: {} is safely ignored since the server is not alive: {}",
              this, JavaUtils.getClassSimpleName(e.getClass()), server, e);
        } else {
          LOG.error("{}: Failed, state={}", this, state, e);
        }
        shutdown();
      }
    } finally {
      // Update leader election completion metric(s).
      electionContext.stop();
      server.getLeaderElectionMetrics().onNewLeaderElectionCompletion();
      lifeCycle.checkStateAndClose(() -> {});
    }
  }
```

首先看submitRequests()方法，该方法是将requestVote请求发给其他RaftPeer，这里并不涉及自身，形参中RaftPeer列表的名字是others，该方法返回发起requestVote的投票的数量

```java
private int submitRequests(Phase phase, long electionTerm, TermIndex lastEntry,
      Collection<RaftPeer> others, Executor voteExecutor) {
    int submitted = 0;
    for (final RaftPeer peer : others) {
        final RequestVoteRequestProto r = ServerProtoUtils.toRequestVoteRequestProto(
            server.getMemberId(), peer.getId(), electionTerm, lastEntry, phase == Phase.PRE_VOTE);
        voteExecutor.submit(() -> server.getServerRpc().requestVote(r));
        submitted++;
    }
    return submitted;
}
```

waitForResult方法比较长

```java
private ResultAndTerm waitForResults(Phase phase, long electionTerm, int submitted,
      RaftConfigurationImpl conf, Executor voteExecutor) throws InterruptedException {
    final Timestamp timeout = Timestamp.currentTime().addTime(server.getRandomElectionTimeout());
    final Map<RaftPeerId, RequestVoteReplyProto> responses = new HashMap<>();
    final List<Exception> exceptions = new ArrayList<>();
    int waitForNum = submitted;
    Collection<RaftPeerId> votedPeers = new ArrayList<>();
    Collection<RaftPeerId> rejectedPeers = new ArrayList<>();
    Set<RaftPeerId> higherPriorityPeers = getHigherPriorityPeers(conf);

    while (waitForNum > 0 && shouldRun(electionTerm)) {
        final TimeDuration waitTime = timeout.elapsedTime().apply(n -> -n);
        if (waitTime.isNonPositive()) {
            if (conf.hasMajority(votedPeers, server.getId())) {
                // if some higher priority peer did not response when timeout, but candidate get majority, candidate pass vote
                return logAndReturn(phase, Result.PASSED, responses, exceptions);
            } else {
                return logAndReturn(phase, Result.TIMEOUT, responses, exceptions);
            }
        }

        try {
            final Future<RequestVoteReplyProto> future = voteExecutor.poll(waitTime);
            if (future == null) {
                continue; // poll timeout, continue to return Result.TIMEOUT
            }

            final RequestVoteReplyProto r = future.get();
            final RaftPeerId replierId = RaftPeerId.valueOf(r.getServerReply().getReplyId());
            final RequestVoteReplyProto previous = responses.putIfAbsent(replierId, r);
            if (previous != null) {
                if (LOG.isWarnEnabled()) {
                    LOG.warn("{} received duplicated replies from {}, the 2nd reply is ignored: 1st={}, 2nd={}",
                             this, replierId,
                             ServerStringUtils.toRequestVoteReplyString(previous),
                             ServerStringUtils.toRequestVoteReplyString(r));
                }
                continue;
            }
            if (r.getShouldShutdown()) {
                return logAndReturn(phase, Result.SHUTDOWN, responses, exceptions);
            }
            if (r.getTerm() > electionTerm) {
                return logAndReturn(phase, Result.DISCOVERED_A_NEW_TERM, responses, exceptions, r.getTerm());
            }

            // If any peer with higher priority rejects vote, candidate can not pass vote
            if (!r.getServerReply().getSuccess() && higherPriorityPeers.contains(replierId)) {
                return logAndReturn(phase, Result.REJECTED, responses, exceptions);
            }

            // remove higher priority peer, so that we check higherPriorityPeers empty to make sure
            // all higher priority peers have replied
            higherPriorityPeers.remove(replierId);

            if (r.getServerReply().getSuccess()) {
                votedPeers.add(replierId);
                // If majority and all peers with higher priority have voted, candidate pass vote
                if (higherPriorityPeers.size() == 0 && conf.hasMajority(votedPeers, server.getId())) {
                    return logAndReturn(phase, Result.PASSED, responses, exceptions);
                }
            } else {
                rejectedPeers.add(replierId);
                if (conf.majorityRejectVotes(rejectedPeers)) {
                    return logAndReturn(phase, Result.REJECTED, responses, exceptions);
                }
            }
        } catch(ExecutionException e) {
            LogUtils.infoOrTrace(LOG, () -> this + " got exception when requesting votes", e);
            exceptions.add(e);
        }
        waitForNum--;
    }
    // received all the responses
    if (conf.hasMajority(votedPeers, server.getId())) {
        return logAndReturn(phase, Result.PASSED, responses, exceptions);
    } else {
        return logAndReturn(phase, Result.REJECTED, responses, exceptions);
    }
}
```



首先涉及到的方法是askForVotes()方法

```java
private boolean askForVotes(Phase phase) throws InterruptedException, IOException {
    for(int round = 0; shouldRun(); round++) {
        final long electionTerm;
        final RaftConfigurationImpl conf;
        synchronized (server) {
            if (!shouldRun()) {
                return false;
            }
            final ConfAndTerm confAndTerm = server.getState().initElection(phase);
            electionTerm = confAndTerm.getTerm();
            conf = confAndTerm.getConf();
        }

        LOG.info("{} {} round {}: submit vote requests at term {} for {}", this, phase, round, electionTerm, conf);
        final ResultAndTerm r = submitRequestAndWaitResult(phase, conf, electionTerm);
        LOG.info("{} {} round {}: result {}", this, phase, round, r);

        synchronized (server) {
            if (!shouldRun(electionTerm)) {
                return false; // term already passed or this should not run anymore.
            }

            switch (r.getResult()) {
                case PASSED:
                    return true;
                case NOT_IN_CONF:
                case SHUTDOWN:
                    server.getRaftServer().close();
                    server.getStateMachine().event().notifyServerShutdown(server.getRoleInfoProto());
                    return false;
                case TIMEOUT:
                    continue; // should retry
                case REJECTED:
                case DISCOVERED_A_NEW_TERM:
                    final long term = r.maxTerm(server.getState().getCurrentTerm());
                    server.changeToFollowerAndPersistMetadata(term, false, r);
                    return false;
                default: throw new IllegalArgumentException("Unable to process result " + r.result);
            }
        }
    }
    return false;
}
```
