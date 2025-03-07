## 1. 前言

在看Ozone的时候，突然意识到不清楚Ozone是怎么使用Ratis的，回到Ratis中，从CounterServer出发，看一下user application是怎么使用ratis的.

代码存在于CounterServer中，可以看到入口类是RaftServer，通过调用RaftServer.start()方法启动了服务类

```java
private final RaftServer server;

public CounterServer(RaftPeer peer, File storageDir) throws IOException {
    //create a property object
    final RaftProperties properties = new RaftProperties();

    //set the storage directory (different for each peer) in the RaftProperty object
    RaftServerConfigKeys.setStorageDir(properties, Collections.singletonList(storageDir));

    //set the port (different for each peer) in RaftProperty object
    final int port = NetUtils.createSocketAddr(peer.getAddress()).getPort();
    GrpcConfigKeys.Server.setPort(properties, port);

    //create the counter state machine which holds the counter value
    final CounterStateMachine counterStateMachine = new CounterStateMachine();

    //build the Raft server
    this.server = RaftServer.newBuilder()
        .setGroup(Constants.RAFT_GROUP)
        .setProperties(properties)
        .setServerId(peer.getId())
        .setStateMachine(counterStateMachine)
        .build();
}

public void start() throws IOException {
    server.start();
  }
```

首先看一下RaftServer类是怎么构造的，以及具体的实现类是什么。以下代码来自RaftServer接口

```java
public RaftServer build() throws IOException {
      return newRaftServer(
          serverId,
          group,
          option,
          Objects.requireNonNull(stateMachineRegistry , "Neither 'stateMachine' nor 'setStateMachineRegistry' " +
              "is initialized."),
          Objects.requireNonNull(properties, "The 'properties' field is not initialized."),
          parameters);
}

private static RaftServer newRaftServer(RaftPeerId serverId, RaftGroup group, RaftStorage.StartupOption option,
        StateMachine.Registry stateMachineRegistry, RaftProperties properties, Parameters parameters)
        throws IOException {
      try {
        return (RaftServer) NEW_RAFT_SERVER_METHOD.invoke(null,
            serverId, group, option, stateMachineRegistry, properties, parameters);
      } catch (IllegalAccessException e) {
        throw new IllegalStateException("Failed to build " + serverId, e);
      } catch (InvocationTargetException e) {
        throw IOUtils.asIOException(e.getCause());
      }
}
```

NEW_RAFT_SERVER_METHOD是Method类型，为ServerImplUtils.newRaftServer()方法

```java
  public static RaftServerProxy newRaftServer(
      RaftPeerId id, RaftGroup group, RaftStorage.StartupOption option, StateMachine.Registry stateMachineRegistry,
      RaftProperties properties, Parameters parameters) throws IOException {
    RaftServer.LOG.debug("newRaftServer: {}, {}", id, group);
    if (group != null && !group.getPeers().isEmpty()) {
      Preconditions.assertNotNull(id, "RaftPeerId %s is not in RaftGroup %s", id, group);
      Preconditions.assertNotNull(group.getPeer(id), "RaftPeerId %s is not in RaftGroup %s", id, group);
    }
    final RaftServerProxy proxy = newRaftServer(id, stateMachineRegistry, properties, parameters);
    proxy.initGroups(group, option);
    return proxy;
  }
```

最终构建RaftServer对象是通过 **final RaftServerProxy proxy = newRaftServer(id, stateMachineRegistry, properties, parameters);** 进行构造的，也就是说RaftServer的实现类是在RaftServerProxy中

在RaftServerProxy的构造器中，只是对实例变量进行赋值，并没有调用任何变量的start()方法

```java
RaftServerProxy(RaftPeerId id, StateMachine.Registry stateMachineRegistry,
    RaftProperties properties, Parameters parameters) {
    this.properties = properties;
    this.stateMachineRegistry = stateMachineRegistry;

    final RpcType rpcType = RaftConfigKeys.Rpc.type(properties, LOG::info);
    this.factory = ServerFactory.cast(rpcType.newFactory(parameters));

    this.serverRpc = factory.newRaftServerRpc(this);

    this.id = id != null? id: RaftPeerId.valueOf(getIdStringFrom(serverRpc));
    this.lifeCycle = new LifeCycle(this.id + "-" + JavaUtils.getClassSimpleName(getClass()));

    this.dataStreamServerRpc = new DataStreamServerImpl(this, parameters).getServerRpc();

    this.executor = ConcurrentUtils.newThreadPoolWithMax(
        RaftServerConfigKeys.ThreadPool.proxyCached(properties),
        RaftServerConfigKeys.ThreadPool.proxySize(properties),
        id + "-impl");

    final TimeDuration rpcSlownessTimeout = RaftServerConfigKeys.Rpc.slownessTimeout(properties);
    final TimeDuration leaderStepDownWaitTime = RaftServerConfigKeys.LeaderElection.leaderStepDownWaitTime(properties);
    this.pauseMonitor = new JvmPauseMonitor(id,
                                            extraSleep -> handleJvmPause(extraSleep, rpcSlownessTimeout, leaderStepDownWaitTime));
}
```

核心逻辑在RaftServerProxy.start()方法中，这里是对rpc等相关服务进行了启动

```java
@Override
public void start() throws IOException {
    lifeCycle.startAndTransition(this::startImpl, IOException.class);
}
```

start()方法调用了内部的startImpl()方法

```java
private void startImpl() throws IOException {
    //类名::实例方法，形参的第一个参数作为调用者，其他参数作为实参传递给实例方法
    //这里呢，就是对于每一个RaftServerImpl，调用其start()方法，使用executor调用
    ConcurrentUtils.parallelForEachAsync(getImpls(), RaftServerImpl::start, executor).join();

    LOG.info("{}: start RPC server", getId());
    getServerRpc().start();
    getDataStreamServerRpc().start();

    pauseMonitor.start();
}
```

这里会调用RaftServerImpl.start()方法，注意这里是RaftServerImpl对象的集合，同时调用getServerRpc().start()方法。这里先忽略掉getDataStreamServerRpc().start()方法，这是Ratis-Streaming相关的

### 1. RaftServerImpl.start()方法



### 2. RaftServerRpc.start()方法

getRaftServerRpc()返回GrpcService对象，GrpcService在构造器中，添加了grpc相关的server

```java
private GrpcService(RaftServer raftServer, Supplier<RaftPeerId> idSupplier,
      String adminHost, int adminPort, GrpcTlsConfig adminTlsConfig,
      String clientHost, int clientPort, GrpcTlsConfig clientTlsConfig,
      String serverHost, int serverPort, GrpcTlsConfig serverTlsConfig,
      SizeInBytes grpcMessageSizeMax, SizeInBytes appenderBufferSize,
      SizeInBytes flowControlWindow,TimeDuration requestTimeoutDuration,
      boolean useSeparateHBChannel) {
    super(idSupplier, id -> new PeerProxyMap<>(id.toString(),
        p -> new GrpcServerProtocolClient(p, flowControlWindow.getSizeInt(),
            requestTimeoutDuration, serverTlsConfig, useSeparateHBChannel)));
    if (appenderBufferSize.getSize() > grpcMessageSizeMax.getSize()) {
      throw new IllegalArgumentException("Illegal configuration: "
          + RaftServerConfigKeys.Log.Appender.BUFFER_BYTE_LIMIT_KEY + " = " + appenderBufferSize
          + " > " + GrpcConfigKeys.MESSAGE_SIZE_MAX_KEY + " = " + grpcMessageSizeMax);
    }

    final RaftProperties properties = raftServer.getProperties();
    this.executor = ConcurrentUtils.newThreadPoolWithMax(
        GrpcConfigKeys.Server.asyncRequestThreadPoolCached(properties),
        GrpcConfigKeys.Server.asyncRequestThreadPoolSize(properties),
        getId() + "-request-");
    this.clientProtocolService = new GrpcClientProtocolService(idSupplier, raftServer, executor);

    this.serverInterceptor = new MetricServerInterceptor(
        idSupplier,
        JavaUtils.getClassSimpleName(getClass()) + "_" + serverPort
    );

    final boolean separateAdminServer = adminPort != serverPort && adminPort > 0;
    final boolean separateClientServer = clientPort != serverPort && clientPort > 0;

    final NettyServerBuilder serverBuilder =
        startBuildingNettyServer(serverHost, serverPort, serverTlsConfig, grpcMessageSizeMax, flowControlWindow);
    serverBuilder.addService(ServerInterceptors.intercept(
        new GrpcServerProtocolService(idSupplier, raftServer), serverInterceptor));
    if (!separateAdminServer) {
      addAdminService(raftServer, serverBuilder);
    }
    if (!separateClientServer) {
      addClientService(serverBuilder);
    }

    final Server server = serverBuilder.build();
    servers.put(GrpcServerProtocolService.class.getSimpleName(), server);
    addressSupplier = newAddressSupplier(serverPort, server);

    if (separateAdminServer) {
      final NettyServerBuilder builder =
          startBuildingNettyServer(adminHost, adminPort, adminTlsConfig, grpcMessageSizeMax, flowControlWindow);
      addAdminService(raftServer, builder);
      final Server adminServer = builder.build();
      servers.put(GrpcAdminProtocolService.class.getName(), adminServer);
      adminServerAddressSupplier = newAddressSupplier(adminPort, adminServer);
    } else {
      adminServerAddressSupplier = addressSupplier;
    }

    if (separateClientServer) {
      final NettyServerBuilder builder =
          startBuildingNettyServer(clientHost, clientPort, clientTlsConfig, grpcMessageSizeMax, flowControlWindow);
      addClientService(builder);
      final Server clientServer = builder.build();
      servers.put(GrpcClientProtocolService.class.getName(), clientServer);
      clientServerAddressSupplier = newAddressSupplier(clientPort, clientServer);
    } else {
      clientServerAddressSupplier = addressSupplier;
    }
}
```

可以看到，添加了

* GrpcServerProtocolService
* GrpcAdminProtocolService
* GrpcClientProtocolService

在其start方法中，遍历Server链表，即上述三个Service，分别调用其start()方法

```java
@Override
public void startImpl() {
    for (Server server : servers.values()) {
        try {
            server.start();
        } catch (IOException e) {
            ExitUtils.terminate(1, "Failed to start Grpc server", e, LOG);
        }
        LOG.info("{}: {} started, listening on {}",
                 getId(), JavaUtils.getClassSimpleName(getClass()), server.getPort());
    }
}
```

需要理解的是，RaftServerProxy构造了一个GrpcService对象，并调用其start()方法，该start()方法添加了三个服务。这也从侧面证明了为了避免socket风暴，multi-raft抽象了socket层，即RaftServerProxy。那么在所有的grpc请求方来说，都会加入groupId，这三个服务根据groupId将请求转发到对应的RaftServerImpl对象中

这里以GrpcServerProtocolService中的appendEntries为例进行分析，输入流的逻辑在ServerRequestStreamObserver匿名内部类中的process()方法，这里调用的是RaftServerProxy.appendEntriesAsync(AppendEntriesRequestProto)方法中

```java
@Override
public StreamObserver<AppendEntriesRequestProto> appendEntries(
    StreamObserver<AppendEntriesReplyProto> responseObserver) {
    return new ServerRequestStreamObserver<AppendEntriesRequestProto, AppendEntriesReplyProto>(
        RaftServerProtocol.Op.APPEND_ENTRIES, responseObserver) {
        @Override
        CompletableFuture<AppendEntriesReplyProto> process(AppendEntriesRequestProto request) throws IOException {
            return server.appendEntriesAsync(request);
        }

        @Override
        long getCallId(AppendEntriesRequestProto request) {
            return request.getServerRequest().getCallId();
        }

        @Override
        String requestToString(AppendEntriesRequestProto request) {
            return ServerStringUtils.toAppendEntriesRequestString(request);
        }

        @Override
        String replyToString(AppendEntriesReplyProto reply) {
            return ServerStringUtils.toAppendEntriesReplyString(reply);
        }

        @Override
        boolean replyInOrder(AppendEntriesRequestProto request) {
            return request.getEntriesCount() != 0;
        }

        @Override
        StatusRuntimeException wrapException(Throwable e, AppendEntriesRequestProto request) {
            return GrpcUtil.wrapException(e, getCallId(request), request.getEntriesCount() == 0);
        }
    };
}
```

RaftServerProxy.appendEntriesAsync()方法，根据groupId转发到对应的RaftServerImpl对象，调用其appendEntries(AppendEntriesRequestProto)方法

```java
@Override
public AppendEntriesReplyProto appendEntries(AppendEntriesRequestProto request) throws IOException {
    return getImpl(request.getServerRequest()).appendEntries(request);
}
```

以上，就是raft server作为server端的整体分析，那么其作为client侧呢？比如leader发送appendEntries，candidate发送RequestVote，都是通过什么stub获取到的RpcClientProxy对象呢？

## 2. 华丽的分割线

还是先看appendEntries中的逻辑，这部分逻辑中leader充当客户端的角色，follower充当服务端的角色，leader在收到client的读写操作时，会先构建logEntry对象，将logEntry放入到自己的RaftLog中，并通过GrpcLogAppender发送到各个follower，一个GrpcLogAppender对象对应到一个follower的rpc，其逻辑在GrpcLogAppender$StreamObservers类中

```java
static class StreamObservers {
    private final StreamObserver<AppendEntriesRequestProto> appendLog;
    private final StreamObserver<AppendEntriesRequestProto> heartbeat;

    StreamObservers(GrpcServerProtocolClient client, AppendLogResponseHandler handler, boolean separateHeartbeat) {
        this.appendLog = client.appendEntries(handler, false);
        this.heartbeat = separateHeartbeat? client.appendEntries(handler, true): null;
    }

    void onNext(AppendEntriesRequestProto proto) {
        if (heartbeat != null && proto.getEntriesCount() == 0) {
            heartbeat.onNext(proto);
        } else {
            appendLog.onNext(proto);
        }
    }

    void onCompleted() {
        appendLog.onCompleted();
        Optional.ofNullable(heartbeat).ifPresent(StreamObserver::onCompleted);
    }
}
```

可以看到，核心点在StreamObservers的构造器中，通过GrpcServiceProtocolClient.appendEntries()方法，实际上是构造了对应的Stub对象