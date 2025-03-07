## 1. 前言

根据Grpc.proto中定义

```protobuf
service RaftServerProtocolService {
  rpc requestVote(ratis.common.RequestVoteRequestProto)
      returns(ratis.common.RequestVoteReplyProto) {}

  rpc startLeaderElection(ratis.common.StartLeaderElectionRequestProto)
      returns(ratis.common.StartLeaderElectionReplyProto) {}

  rpc appendEntries(stream ratis.common.AppendEntriesRequestProto)
      returns(stream ratis.common.AppendEntriesReplyProto) {}

  rpc installSnapshot(stream ratis.common.InstallSnapshotRequestProto)
      returns(stream ratis.common.InstallSnapshotReplyProto) {}
}
```

server端处理requestVote的流程在RaftServerImpl中

```java
@Override
  public RequestVoteReplyProto requestVote(RequestVoteRequestProto r) throws IOException {
      final RaftRpcRequestProto request = r.getServerRequest();
      return requestVote(r.getPreVote() ? Phase.PRE_VOTE : Phase.ELECTION,
                         RaftPeerId.valueOf(request.getRequestorId()),
                         ProtoUtils.toRaftGroupId(request.getRaftGroupId()),
                         r.getCandidateTerm(),
                         TermIndex.valueOf(r.getCandidateLastEntry()));
  }
```

该方法调用使用private修饰的同名方法

```java
private RequestVoteReplyProto requestVote(Phase phase,
      RaftPeerId candidateId, RaftGroupId candidateGroupId,
      long candidateTerm, TermIndex candidateLastEntry) throws IOException {
    CodeInjectionForTesting.execute(REQUEST_VOTE, getId(),
                                    candidateId, candidateTerm, candidateLastEntry);
    LOG.info("{}: receive requestVote({}, {}, {}, {}, {})",
             getMemberId(), phase, candidateId, candidateGroupId, candidateTerm, candidateLastEntry);
    assertLifeCycleState(LifeCycle.States.RUNNING);
    assertGroup(candidateId, candidateGroupId);

    boolean shouldShutdown = false;
    final RequestVoteReplyProto reply;
    synchronized (this) {
        // Check life cycle state again to avoid the PAUSING/PAUSED state.
        assertLifeCycleState(LifeCycle.States.RUNNING);

        //这里是构造的context，即VoteContext对象
        final VoteContext context = new VoteContext(this, phase, candidateId);
        final RaftPeer candidate = context.recognizeCandidate(candidateTerm);
        //所以这里是逻辑入口咯，下面首先判断是不是election阶段，我这里要看的是preVote，所以
        final boolean voteGranted = context.decideVote(candidate, candidateLastEntry);
        if (candidate != null && phase == Phase.ELECTION) {
            // change server state in the ELECTION phase
            final boolean termUpdated =
                changeToFollower(candidateTerm, true, false, "candidate:" + candidateId);
            if (voteGranted) {
                state.grantVote(candidate.getId());
            }
            if (termUpdated || voteGranted) {
                state.persistMetadata(); // sync metafile
            }
        }
        //呼应上面注释，这里是preVote的入口，但是问题的关键还是voteGranted是怎么得到的，其判断逻辑跟Phase.ELECTION相同
        if (voteGranted) {
            //这里有一点点奇怪的地方，就是在Follower阶段会收到preVote RPC吗，这里更新lastRpcTime有什么用呢，会阻止其成为
            //Candidate吗，是的，会阻止其成为Candidate
            role.getFollowerState().ifPresent(fs -> fs.updateLastRpcTime(FollowerState.UpdateType.REQUEST_VOTE));
        } else if(shouldSendShutdown(candidateId, candidateLastEntry)) {
            shouldShutdown = true;
        }
        reply = ServerProtoUtils.toRequestVoteReplyProto(candidateId, getMemberId(),
                                                         voteGranted, state.getCurrentTerm(), shouldShutdown);
        if (LOG.isInfoEnabled()) {
            LOG.info("{} replies to {} vote request: {}. Peer's state: {}",
                     getMemberId(), phase, ServerStringUtils.toRequestVoteReplyString(reply), state);
        }
    }
    return reply;
}
```

