## 1. 前言

用来表示RPC的具体实现

其接口方法比较简单

```java
public interface RpcType {
    String name();
    RpcFactory newFactory(Parameters parameters);
}
```

接口内部定义了一个Get接口，用来得到RpcType对象

```java
public interface RpcType {
    interface Get { 
    	RpcType getRpcType();
    }
}
```

RpcFactory只是定义为了RpcType.Get的子接口

```java
public interface RpcFactory extends RpcType.Get {
}
```

## 2. 接口实现类SupportedRpcType

```java
public enum SupportedRpcType implements RpcType  {
    NETTY("org.apache.ratis.netty.NettyFactory"),
    GRPC("org.apache.ratis.grpc.GrpcFactory"),
    HADOOP("org.apache.ratis.hadooprpc.HadoopFactory");
    
    private final String factoryClassName;
    
    SupportedRpcType(String factoryClassName) {
        this.factoryClassName = factoryClassName;
  	}
    
    public static SupportedRpcType valueOfIgnoreCase(String s) {
    	return valueOf(s.toUpperCase());
  	}
    
    @Override
  	public RpcFactory newFactory(Parameters parameters) {
    	final Class<? extends RpcFactory> clazz = ReflectionUtils.getClass(
        	factoryClassName, RpcFactory.class);
    	return ReflectionUtils.newInstance(clazz, ARG_CLASSES, parameters);
    }
}
```

## 3. RpcFactory的子接口及实现类

RpcFactory是RpcType的子接口，没有定义任何api，其存在两个子接口，分别为ClientFactory和ServerFactory

```java
public interface ClientFactory extends RpcFactory {
    RaftClinetRpc newRaftClientRpc(ClientId clientId, RaftProperties properties);
    
    static ClientFactory cast(RpcFactory rpcFactory) {
        if(rpcFactory instanceof ClientFactory) {
            return (ClientFactory) rpcFactory;
        }
        
        throw new ClassCastException();
    }
}
```

ServerFactory的内容同ClientFactory的内容基本一致

```java
public interface ServerFactory extends RpcFactory {
    RaftServerRpc newRaftServerRpc(RaftServer server);
    
    static ServerFactory cast(RpcFactory rpcFactory) {
        //...省略
    }
    
    default LogAppender newLogAppender(RaftServer.Division server, LeaderState state, FollowInfo f){
        return LogAppender.newLogAppenderDefault(server, state, f);
    }
}
```

那么么RpcFactory是顶级的接口，其子类分成了两个分支，为

* ClientFactory
* ServerFactory

分别提供了api，用来生成

* RaftClientRpc
* RaftServerRpc

那么，接下来，将目光放到RaftClientRpc和RaftServerRpc中

## 4. RaftClientRpc和RaftServerRpc

### 1. RaftClientRpc

提供了5个API

```java
public interface RaftClientRpc extends RaftPeer.Add, Closeable {
  /** Async call to send a request. */
  default CompletableFuture<RaftClientReply> sendRequestAsync(RaftClientRequest request) {
    throw new UnsupportedOperationException(getClass() + " does not support this method.");
  }

  /** Async call to send a request. */
  default CompletableFuture<RaftClientReply> sendRequestAsyncUnordered(RaftClientRequest request) {
    throw new UnsupportedOperationException(getClass() + " does not support "
        + JavaUtils.getCurrentStackTraceElement().getMethodName());
  }

  /** Send a request. */
  RaftClientReply sendRequest(RaftClientRequest request) throws IOException;

  /**
   * Handle the given throwable. For example, try reconnecting.
   *
   * @return true if the given throwable is handled; otherwise, the call is an no-op, return false.
   */
  default boolean handleException(RaftPeerId serverId, Throwable t, boolean reconnect) {
    return false;
  }

  /**
   * Determine if the given throwable should be handled. For example, try reconnecting.
   *
   * @return true if the given throwable should be handled; otherwise, return false.
   */
  default boolean shouldReconnect(Throwable t) {
    return IOUtils.shouldReconnect(t);
  }
}
```

其实现类为RaftClientRpcWithProxy类，该类为一个抽象类。

```java
public abstract class RaftClientRpcWithProxy<PROXY extends Closeable>
    implements RaftClientRpc {
  private final PeerProxyMap<PROXY> proxies;

  protected RaftClientRpcWithProxy(PeerProxyMap<PROXY> proxies) {
    this.proxies = proxies;
  }

  public PeerProxyMap<PROXY> getProxies() {
    return proxies;
  }

  @Override
  public void addRaftPeers(Collection<RaftPeer> servers) {
    proxies.addRaftPeers(servers);
  }

  @Override
  public boolean handleException(RaftPeerId serverId, Throwable t, boolean reconnect) {
    return getProxies().handleException(serverId, t, reconnect);
  }

  @Override
  public void close() {
    proxies.close();
  }
}
```

RaftClientRpcWithProxy的实现类的为GrpcClientRpc。可以看到，GrpcClientRpc并没有指定泛型，而是指定实现了RaftclientRpcWithProxy\<GrpcClientProtocolClient>类型。这里需要注意的是，其父类RaftClientRpcWithProxy持有一个PeerProxyMap\<RaftClientProtocolClient>对象，这是一个map对象，用来缓存到很多个raftPeer的GrpcClientProtocolClient对象

```java
public class GrpcClientRpc extends RaftClientRpcWithProxy<GrpcClientProtocolClient> {
    private final ClientId clientId;
    private final maxMessageSize;
}
```

最值得关注的方法是构造器，构造器中在调用super()方法，及其父类构造器时，需要传入一个PeerProxyMap对象，而该PeerProxyMap对象的构造器中需要传入createProxy对象，而这个createProxy对象是关键之处

```java
public GrpcClientRpc(ClientId clientId, RaftProperties properties,
                     GrpcTlsConfig adminTlsConfig, GrpcTlsConfig clientTlsConfig) {
    super(new PeerProxyMap<>(clientId.toString(),
     p -> new GrpcClientProtocolClient(clientId, p, properties, adminTlsConfig, clientTlsConfig)));
    this.clientId = clientId;
    this.maxMessageSize = GrpcConfigKeys.messageSizeMax(properties, LOG::debug).getSizeInt();
  }
```

而接下来实现RaftClientRpc中各种的send()方法都委托给代理对象，及GrpcClientProtocolClient对象去进行操作

```java
  @Override
  public CompletableFuture<RaftClientReply> sendRequestAsync(
      RaftClientRequest request) {
      final RaftPeerId serverId = request.getServerId();
      try {
          final GrpcClientProtocolClient proxy = getProxies().getProxy(serverId);
          // Reuse the same grpc stream for all async calls.
          return proxy.getOrderedStreamObservers().onNext(request);
      } catch (Exception e) {
          return JavaUtils.completeExceptionally(e);
      }
  }

  @Override
  public CompletableFuture<RaftClientReply> sendRequestAsyncUnordered(RaftClientRequest request) {
      final RaftPeerId serverId = request.getServerId();
      try {
          final GrpcClientProtocolClient proxy = getProxies().getProxy(serverId);
          // Reuse the same grpc stream for all async calls.
          return proxy.getUnorderedAsyncStreamObservers().onNext(request);
      } catch (Exception e) {
          LOG.error(clientId + ": Failed " + request, e);
          return JavaUtils.completeExceptionally(e);
      }
  }
```

那么问题来了，GrpcClient对象在哪里使用的呢？在GrpcFactory中

```java
public class GrpcFactory implements ServerFactory, ClientFactory {
    @Override
    public GrpcClientRpc newRaftClientRpc(ClientId clientId, RaftProperties properties) {
        checkPooledByteBufAllocatorUseCacheForAllThreads(LOG::debug);
        return new GrpcClientRpc(clientId, properties,
                                 getAdminTlsConfig(), getClientTlsConfig());
    }
}
```

### 2. RaftServerRpc

提供了六个API

```java
public interface RaftServerRpc extends RaftServerProtocol, RpcType.Get, RaftPeer.Add, Closeable {
  /** Start the RPC service. */
  void start() throws IOException;

  /** @return the address where this RPC server is listening */
  InetSocketAddress getInetSocketAddress();

  /** @return the address where this RPC server is listening for client requests */
  default InetSocketAddress getClientServerAddress() {
    return getInetSocketAddress();
  }
  /** @return the address where this RPC server is listening for admin requests */
  default InetSocketAddress getAdminServerAddress() {
    return getInetSocketAddress();
  }

  /** Handle the given exception.  For example, try reconnecting. */
  void handleException(RaftPeerId serverId, Exception e, boolean reconnect);

  /** The server role changes from leader to a non-leader role. */
  default void notifyNotLeader(RaftGroupId groupId) {
  }
}
```

跟RaftClientRpc类似，也有一个抽象类RaftServerRpcWithProxy，这里的PROXY和PROXIES其实跟RaftClientRpc是一样的逻辑。这里的proxyCreater是一个Function，跟PeerProxyMap中的createPrxoy没有关系，不要误会了！

```java
public abstract class RaftServerRpcWithProxy<PROXY extends Closeable, PROXIES extends PeerProxyMap<PROXY>> implements RaftServerRpc {
      private final Supplier<RaftPeerId> idSupplier;
      private final Supplier<LifeCycle> lifeCycleSupplier;
      private final Supplier<PROXIES> proxiesSupplier;

      protected RaftServerRpcWithProxy(Supplier<RaftPeerId> idSupplier, Function<RaftPeerId, PROXIES> proxyCreater) {
        this.idSupplier = idSupplier;
        this.lifeCycleSupplier = JavaUtils.memoize(
            () -> new LifeCycle(getId() + "-" + JavaUtils.getClassSimpleName(getClass())));
        this.proxiesSupplier = JavaUtils.memoize(() -> proxyCreater.apply(getId()));
      }

      /** @return the server id. */
      public RaftPeerId getId() {
        return idSupplier.get();
      }

      private LifeCycle getLifeCycle() {
        return lifeCycleSupplier.get();
      }

      /** @return the underlying {@link PeerProxyMap}. */
      public PROXIES getProxies() {
        return proxiesSupplier.get();
      }

      @Override
      public void addRaftPeers(Collection<RaftPeer> peers) {
        getProxies().addRaftPeers(peers);
      }

      @Override
      public void handleException(RaftPeerId serverId, Exception e, boolean reconnect) {
        getProxies().handleException(serverId, e, reconnect);
      }

      @Override
      public final void start() throws IOException {
        getLifeCycle().startAndTransition(this::startImpl, IOException.class);
      }

      /** Implementation of the {@link #start()} method. */
      protected abstract void startImpl() throws IOException;

      @Override
      public final void close() throws IOException{
        getLifeCycle().checkStateAndClose(this::closeImpl);
      }

      /** Implementation of the {@link #close()} method. */
      public void closeImpl() throws IOException {
        getProxies().close();
      }
}
```

RaftServerRpcWithProxy的子类为GrpcService，主要关注其构造器中调用super()时传递的proxyCreater对象

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

GprcService使用的是Builder模式，其build()方法的caller为GrpcFactory，对应的为newRaftServerRpc()方法

```java
@Override
  public GrpcService newRaftServerRpc(RaftServer server) {
    checkPooledByteBufAllocatorUseCacheForAllThreads(LOG::info);
    return GrpcService.newBuilder()
        .setServer(server)
        .setAdminTlsConfig(getAdminTlsConfig())
        .setServerTlsConfig(getServerTlsConfig())
        .setClientTlsConfig(getClientTlsConfig())
        .build();
  }
```

### 3. 总结

总的来说

* RaftClientRpcWithProxy持有一个PeerProxyMap\<PROXY>对象，该对象本身是一个map，用来做raftPeer->GrpcClientProtocolClient的缓存
* RaftServerRpcWithProxy持有一个Supplier\<PeerProxyMap\<PROXY>>对象，用来做raftPeer->GrpcClientProtocolService的缓存

## 5. RaftServer接口

### 1. Division接口

用来表示一个RaftServer上指定RaftGroup的一个实例对象

```java
interface Division extends Closeable {
    DivisionProperties properties();
    
    RaftGroupMemeberId getMemberId();
    
    default RaftPeerId getId() {
        return getMemberId().getPeerId();
    }
        
    default RaftPeer getPeer() {
        return Optional.ofNullable(getRaftConf().getPeer(getId()))
            .orElseGet(() -> getRaftServer().getPeer());
    }
    
    DivisionInfo getInfo();
    
    default RaftGroup getGroup() {
      Collection<RaftPeer> allFollowerPeers =
          getRaftConf().getAllPeers(RaftProtos.RaftPeerRole.FOLLOWER);
      Collection<RaftPeer> allListenerPeers =
          getRaftConf().getAllPeers(RaftProtos.RaftPeerRole.LISTENER);
      Iterable<RaftPeer> peers = Iterables.concat(allFollowerPeers, allListenerPeers);
      return RaftGroup.valueOf(getMemberId().getGroupId(), peers);
    }
    
    RaftConfiguration getRaftConf();

    /** @return the {@link RaftServer} containing this division. */
    RaftServer getRaftServer();

    /** @return the {@link RaftServerMetrics} for this division. */
    RaftServerMetrics getRaftServerMetrics();

    /** @return the {@link StateMachine} for this division. */
    StateMachine getStateMachine();

    /** @return the raft log of this division. */
    RaftLog getRaftLog();

    /** @return the storage of this division. */
    RaftStorage getRaftStorage();

    /** @return the commit information of this division. */
    Collection<CommitInfoProto> getCommitInfos();

    /** @return the retry cache of this division. */
    RetryCache getRetryCache();

    /** @return the data stream map of this division. */
    DataStreamMap getDataStreamMap();

    /** @return the internal {@link RaftClient} of this division. */
    RaftClient getRaftClient();

    @Override
    void close();
}
```

### 2. RaftServer接口定义

```java
public interface RaftServer extends Closeable, RpcType.Get,
    RaftServerProtocol, RaftServerAsynchronousProtocol,
    RaftClientProtocol, RaftClientAsynchronousProtocol,
    AdminProtocol, AdminAsynchronousProtocol {
   
  RaftPeerId getId();

  /** @return the {@link RaftPeer} for this server. */
  RaftPeer getPeer();

  /** @return the group IDs the server is part of. */
  Iterable<RaftGroupId> getGroupIds();

  /** @return the groups the server is part of. */
  Iterable<RaftGroup> getGroups() throws IOException;

  Division getDivision(RaftGroupId groupId) throws IOException;

  /** @return the server properties. */
  RaftProperties getProperties();

  /** @return the rpc service. */
  RaftServerRpc getServerRpc();

  /** @return the data stream rpc service. */
  DataStreamServerRpc getDataStreamServerRpc();

  /** @return the {@link RpcType}. */
  default RpcType getRpcType() {
    return getFactory().getRpcType();
  }

  /** @return the factory for creating server components. */
  ServerFactory getFactory();

  /** Start this server. */
  void start() throws IOException;

  LifeCycle.State getLifeCycleState();

  /** @return a {@link Builder}. */
  static Builder newBuilder() {
    return new Builder();
  }
}
```

### 3. DivisionInfo接口

用来藐视一个raft server Division对象

```java
public interface DivisionInfo {
    RaftPeerRole getCurrentRole();
    
    default boolean isFollower() {
        return getCurrentRole() == RaftPeerRole.FOLLOWER;
    }
    
    default boolean isCandidate() {
        return getCurrentRole() == RaftPeerRole.CANDIDATE;
    }

    /** Is this server division currently the leader? */
    default boolean isLeader() {
        return getCurrentRole() == RaftPeerRole.LEADER;
    }

    default boolean isListener() {
        return getCurrentRole() == RaftPeerRole.LISTENER;
    }

    boolean isLeaderReady();
    
    RaftPeerId getLeaderId();
    
    LifeCycle.State getLifeCycleState();
    
    default boolean isAlive() {
        return !getLifeCycleState().isClosingOrClosed();
    }
    
    RoleInfoProto getRoleInfoProto();
    
    long getCurrentTerm();
    
    long getLastAppliedIndex();
    
    long[] getFollowerNextIndicies();
}
```

