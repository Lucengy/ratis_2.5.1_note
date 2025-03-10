## 1. 前言

在RaftServerImpl中，持有一个RoleInfo对象，使用final进行修饰，并且在构造器中，通过RaftPeerId实例变量进行赋值

```java
private final RaftRoleInfo role;
RaftServerImpl(RaftGroup group, StateMachine stateMachine, RaftServerProxy wproxy, ...) {
    final RaftPeerId id = proxy.getId();
    // ...
    this.role = new RoleInfo();
    //...
}
```

有关RoleInfo的设置，RaftServerImpl也提供了setRole方法的，但是这里的setRole方法使用的private进行修饰

```java
private void setRole(RaftPeerRole newRole, Object reason) {
    //LOG
    this.role.transitionRole(newRole);
}

void transitionRole(RaftPeerRole newRole) {
    this.role = newRole;
    this.transitionTime.set(Timestamp.currentTime());
}
```

有关setRole方法，首先是startAwsPeer方法，根据JavaDoc，这里是当this peer属于current configuration时进行调用的

```java
private void startAsPeer(RaftPeerRole newRole) {
    Object reason = "";
    if (newRole == RaftPeerRole.FOLLOWER) {
        reason = "startAsFollower";
        setRole(RaftPeerRole.FOLLOWER, reason);
    } else if (newRole == RaftPeerRole.LISTENER) {
        reason = "startAsListener";
        setRole(RaftPeerRole.LISTENER, reason);
    } else {
        throw new IllegalArgumentException("Unexpected role " + newRole);
    }
    role.startFollowerState(this, reason);

    lifeCycle.transition(RUNNING);
}
```

还有三个位置使用setRole方法，分别是changeToLeader，changeToCandidate，changToFollower方法。

但是在RoleInfo.transitionRole()方法中，并没有额外的逻辑，只是简单的对其中的RaftPeerRole实例变量进行赋值，并且重置了transitionTime这个实例变量，用来标识成为当前角色的起始时间

除去setRole方法，还有其他位置直接使用role实例变量，通过调用其实例方法，来执行相应的逻辑代码。

这里以RaftServerImpl.start()方法为入口，在其中，调用了实例方法startAsPeer(RaftPeerRole.FOLLOWER)方法。而在startAsPeer()方法中，除去调用了setRole()方法外，还调用了role.startFollowerState()方法。

在startFollowerState()方法中，构造了新的FollowerState对象，并调用其start()方法

在FollowerState对象中，经过一定的election timeout，调用RaftServerImpl.changeToCandidate()方法，将RaftPeer的状态从Follower改为Candidate

```java
synchronized void changeToCandidate(boolean forceStartLeaderElection) {
    Preconditions.assertTrue(getInfo().isFollower());
    role.shutdownFollowerState();
    setRole(RaftPeerRole.CANDIDATE, "changeToCandidate");
    if (state.shouldNotifyExtendedNoLeader()) {
        stateMachine.followerEvent().notifyExtendedNoLeader(getRoleInfoProto());
    }
    // start election
    role.startLeaderElection(this, forceStartLeaderElection);
}
```

changeToCandidate()方法中

* 调用role.shutdownFollowerState()方法停止FollowerState线程
* 调用setRole方法，修改RoleInfo的状态信息
* 调用RoleInfo对象的startLeaderElection()方法，开启投票

在RoleInfo的startLeaderElection方法中，构建新的LeaderElection对象，并调用其start()方法

## 2. 代码

根据proto文件，Role of raft peer 包含以下几种角色

```protobuf
enum RaftPeerRole {
  LEADER = 0;
  CANDIDATE = 1;
  FOLLOWER = 2;
  LISTENER = 3;
}
```



```java
class RoleInfo {
  private final RaftPeerId id;
  private volatile RaftPeerRole role;
  /** Used when the peer is leader */
  private final AtomicReference<LeaderStateImpl> leaderState = new AtomicReference<>();
  /** Used when the peer is follower, to monitor election timeout */
  private final AtomicReference<FollowerState> followerState = new AtomicReference<>();
  /** Used when the peer is candidate, to request votes from other peers */
  private final AtomicReference<LeaderElection> leaderElection = new AtomicReference<>();
  private final AtomicBoolean pauseLeaderElection = new AtomicBoolean(false);

  private final AtomicReference<Timestamp> transitionTime;
}
```

这里需要重点关注的是transitionTime实例变量，在构造器中，transitionTime被初始化为当前时间

```java
RoleInfo(RaftPeerId id) {
    this.id = id;
    this.transitionTime = new AtomicReference<>(Timestamp.currentTime());
}
```

当raft peer的角色，即RaftPeerRole发生改变时，需要对transitionTime重新赋值

```java
void transitionRole(RaftPeerRole newRole) {
    this.role = newRole;
    this.transitionTime.set(Timestamp.currentTime());
}
```

getRoleElapsedTimeMs()方法用来返回角色持续时间，即当前时间减去transitionTime，单位为ms

```java
long getRoleElapsedTimeMs() {
    return transitionTime.get().elapsedTimeMs();
}
```

剩下的方法放到一起进行说明

```java
class RoleInfo {
    private <T> T updateAndGet(AtomicReference<T> ref, T current) {
        final T updated = ref.updateAndGet(previous -> previous != null? previous: current);
        Preconditions.assertTrue(updated == current, "previous != null");
        return updated;
    }
    
    //这里有一点需要关注的时，leadserStateImpl.start()方法返回的是LogEntryProto对象
    LogEntryProto startLeaderState(RaftServerImpl server) {
        return updateAndGet(leaderState, new LeaderStateImpl(server)).start();
    }
    
    void shutdownLeaderState(boolean allowNull) {
        final LeaderStateImpl leader = leaderState.getAndSet(null);
        if (leader == null) {
          if (!allowNull) {
            throw new NullPointerException("leaderState == null");
          }
        } else {
          LOG.info("{}: shutdown {}", id, leader);
          leader.stop();
        }
        // TODO: make sure that StateMachineUpdater has applied all transactions that have context
    }
    
    void startFollowerState(RaftServerImpl server, Object reason) {
        updateAndGet(followerState, new FollowerState(server, reason)).start();
    }
    
    void shutdownFollowerState() {
        final FollowerState follower = followerState.getAndSet(null);
        if (follower != null) {
          LOG.info("{}: shutdown {}", id, follower);
          follower.stopRunning();
          follower.interrupt();
        }
    }
    
    void startLeaderElection(RaftServerImpl server, boolean force) {
        if (pauseLeaderElection.get()) {
            return;
        }
        updateAndGet(leaderElection, new LeaderElection(server, force)).start();
    }
    
    void setLeaderElectionPause(boolean pause) {
        pauseLeaderElection.set(pause);
    }
    
    void shutdownLeaderElection() {
        final LeaderElection election = leaderElection.getAndSet(null);
        if (election != null) {
            LOG.info("{}: shutdown {}", id, election);
            election.shutdown();
            // no need to interrupt the election thread
        }
    }
}
```

