## 1. 前言

RaftServerImpl是核心类，每个RaftServer持有多个RaftServerImpl对象，每个RaftServerImpl对象属于一个RaftGroup

有关Division、RaftServerProtocol、RaftServerrAsynchonousProtocol、RaftClientProtocol、RaftClientAsynchronousProtocol接口在前文已经描述，这里就不再赘述。需要提一点的是，相较于RaftServer类，这里并没没有实现Admin相关的接口，但是却实现了Admin相关的rpc的方法，也是amazing

## 2. Info

RaftServerImpl在地位上跟Division是对等的，那么其内部类Info实现了DivisionInfo接口

```java
class Info implements DivisionInfo {
    
}
```

## 3. RaftServerImpl实现

### 1. 实例变量

```java
  //这是每个RaftServer的代理类，每个proxy对象持有多个RaftServerImpl对象
  private final RaftServerProxy proxy; 
  //stateMachine跟RaftServerImpl是同一层面的
  private final StateMachine stateMachine;
  
  private final Info info =  new Info();

  private final DivisionProperties divisionProperties;
  private final TimeDuration leaderStepDownWaitTime;
  private final TimeDuration sleepDeviationThreshold;

  private final LifeCycle lifeCycle;
  //ServerState，目前还不知道是跟RaftServerProxy对等还是跟RaftServerImpl对等
  private final ServerState state;
  //当前RaftServerImpl在自己RaftGroup中的角色
  private final RoleInfo role;

  //这个是Streaming Ratis相关的对象
  private final DataStreamMap dataStreamMap;

  //持有一个RaftClient对象？？？跟Ratis Streaming相关 ratis-1178
  private final MemoizedSupplier<RaftClient> raftClient;

  private final RetryCacheImpl retryCache;
  private final CommitInfoCache commitInfoCache = new CommitInfoCache();

  private final RaftServerJmxAdapter jmxAdapter;
  private final LeaderElectionMetrics leaderElectionMetrics;
  private final RaftServerMetricsImpl raftServerMetrics;

  // To avoid append entry before complete start() method
  // For example, if thread1 start(), but before thread1 startAsFollower(), thread2 receive append entry
  // request, and change state to RUNNING by lifeCycle.compareAndTransition(STARTING, RUNNING),
  // then thread1 execute lifeCycle.transition(RUNNING) in startAsFollower(),
  // So happens IllegalStateException: ILLEGAL TRANSITION: RUNNING -> RUNNING,
  private final AtomicBoolean startComplete;

  private final TransferLeadership transferLeadership;
  private final SnapshotManagementRequestHandler snapshotRequestHandler;
  private final SnapshotInstallationHandler snapshotInstallationHandler;

  private final ExecutorService serverExecutor;
  private final ExecutorService clientExecutor;

  private final AtomicBoolean firstElectionSinceStartup = new AtomicBoolean(true);
```

### 2. 构造器

在构造器中，通过

**this.state = new ServerState(id, group, stateMachine, this, option, properties);**

可以看出，ServerState跟RaftServerImpl是同一层面的概念。先看3_ServerState.md

```java
RaftServerImpl(RaftGroup group, StateMachine stateMachine, RaftServerProxy proxy, RaftStorage.StartupOption option)
      throws IOException {
    final RaftPeerId id = proxy.getId();
    LOG.info("{}: new RaftServerImpl for {} with {}", id, group, stateMachine);
    this.lifeCycle = new LifeCycle(id);
    this.stateMachine = stateMachine;
    this.role = new RoleInfo(id);

    final RaftProperties properties = proxy.getProperties();
    this.divisionProperties = new DivisionPropertiesImpl(properties);
    leaderStepDownWaitTime = RaftServerConfigKeys.LeaderElection.leaderStepDownWaitTime(properties);
    this.sleepDeviationThreshold = RaftServerConfigKeys.sleepDeviationThreshold(properties);
    this.proxy = proxy;

    this.state = new ServerState(id, group, stateMachine, this, option, properties);
    this.retryCache = new RetryCacheImpl(properties);
    this.dataStreamMap = new DataStreamMapImpl(id);

    this.jmxAdapter = new RaftServerJmxAdapter();
    this.leaderElectionMetrics = LeaderElectionMetrics.getLeaderElectionMetrics(
        getMemberId(), state::getLastLeaderElapsedTimeMs);
    this.raftServerMetrics = RaftServerMetricsImpl.computeIfAbsentRaftServerMetrics(
        getMemberId(), () -> commitInfoCache::get, retryCache::getStatistics);

    this.startComplete = new AtomicBoolean(false);

    this.raftClient = JavaUtils.memoize(() -> RaftClient.newBuilder()
        .setRaftGroup(group)
        .setProperties(getRaftServer().getProperties())
        .build());

    this.transferLeadership = new TransferLeadership(this);
    this.snapshotRequestHandler = new SnapshotManagementRequestHandler(this);
    this.snapshotInstallationHandler = new SnapshotInstallationHandler(this, properties);

    this.serverExecutor = ConcurrentUtils.newThreadPoolWithMax(
        RaftServerConfigKeys.ThreadPool.serverCached(properties),
        RaftServerConfigKeys.ThreadPool.serverSize(properties),
        id + "-server");
    this.clientExecutor = ConcurrentUtils.newThreadPoolWithMax(
        RaftServerConfigKeys.ThreadPool.clientCached(properties),
        RaftServerConfigKeys.ThreadPool.clientSize(properties),
        id + "-client");
  }
```

