## 1. 前言

这里涉及到的类有

* RaftPeer
* LifeCycle
* PeerProxyMap
* RaftClientRpcWithProxy

## 2. RaftPeer

```
A RaftPeer contains the information of a server.
```

RaftPeer对象代表一台机器，描述里还提到了**The objects of this class are immutable.**关于immutable类的定义为，使用final修饰符修饰class和field，这个类中需要了解的点有两个：一个是内部的Add接口，一个是RaftPeerProto message的定义

1. Add接口

   ```java
     public interface Add {
       /** Add the given peers. */
       void addRaftPeers(Collection<RaftPeer> peers);
   
       /** Add the given peers. */
       default void addRaftPeers(RaftPeer... peers) {
         addRaftPeers(Arrays.asList(peers));
       }
     }
   ```

2. RaftPeerProto 

   ```protobuf
   message RaftPeerProto {
     bytes id = 1;      // id of the peer
     string address = 2; // e.g. address of the RPC server
     uint32 priority = 3; // priority of the peer
     string dataStreamAddress = 4; // address of the data stream server
     string clientAddress = 5; // address of the client RPC server
     string adminAddress = 6; // address of the admin RPC server
     optional RaftPeerRole startupRole = 7; // peer start up role
   }
   ```

2. RaftPeerProto

   ```protobuf
   message RaftPeerProto {
     bytes id = 1;      // id of the peer
     string address = 2; // e.g. address of the RPC server
     uint32 priority = 3; // priority of the peer
     string dataStreamAddress = 4; // address of the data stream server
     string clientAddress = 5; // address of the client RPC server
     string adminAddress = 6; // address of the admin RPC server
     optional RaftPeerRole startupRole = 7; // peer start up role
   }
   ```

## 3. LifeCycle

代表一台机器的生命周期，以下为其状态机示意图

![1735700670094](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1735700670094.png)

这里需要了解的是enum State中的static初始化模块

```java
    static {
      final Map<State, List<State>> predecessors = new EnumMap<>(State.class);
      put(NEW,       predecessors, STARTING);
      put(STARTING,  predecessors, NEW, PAUSED);
      put(RUNNING,   predecessors, STARTING);
      put(PAUSING,   predecessors, RUNNING);
      put(PAUSED,    predecessors, PAUSING);
      put(EXCEPTION, predecessors, STARTING, PAUSING, RUNNING);
      put(CLOSING,   predecessors, STARTING, RUNNING, PAUSING, PAUSED, EXCEPTION);
      put(CLOSED,    predecessors, NEW, CLOSING);

      PREDECESSORS = Collections.unmodifiableMap(predecessors);
    }
```

存放的key是各个状态，value是一个List，代表的是可以转变成key的直接前驱状态

需要关注的方法有startAndTransition()方法，其是先状态改为Starting，然后执行实参中的方法，再将状态改为Running

```java
  public final <T extends Throwable> void startAndTransition(
      CheckedRunnable<T> startImpl, Class<? extends Throwable>... exceptionClasses)
      throws T {
    transition(State.STARTING);
    try {
      startImpl.run();
      transition(State.RUNNING);
    } catch (Throwable t) {
      transition(ReflectionUtils.isInstance(t, exceptionClasses)?
          State.NEW: State.EXCEPTION);
      throw t;
    }
  }
```

## 4. PeerProxyMap

```
/** A map from peer id to peer and its proxy. */
```

这里下需要理解的泛型PROXY，目前了解到的，或者可以理解为特指RaftClientProtocolClient，那么这个类封装的就是peerID，到peer以及其RaftClientProtocolClient的关系。其中，使用内部类PeerAndProxy，将RaftPeer和RaftClientProtocolClient封装到一起。在PeerProxyMap中，使用一个Map\<RaftPeerId, PeerAndProxy>作为方法的操作对象，对外提供服务

首先，了解其内部类，即PeerAndProxy的定义

```java
    private final RaftPeer peer;
    private volatile PROXY proxy = null;
    private final LifeCycle lifeCycle;
```

首先要看的是其构造函数

```java
    PeerAndProxy(RaftPeer peer) {
      this.peer = peer;
      this.lifeCycle = new LifeCycle(peer);
    }
```

构造函数中并没有对RaftClientProtocolClient对象进行赋值，这里的RaftClientProtocolClient对象是核心属性，对其的赋值在getProxy()方法中，getProxy()方法使用了常见的double check单例模式。createProxy为函数式接口，是外部类PeerProxyMap中的实例变量。

setNullProxyAndClose()方法，将RaftClientProtocolClient设置为null，同时返回原RaftClientProtocolClient对象。

```java
    PROXY getProxy() throws IOException {
        if (proxy == null) {
        synchronized (this) {
          if (proxy == null) {
            final LifeCycle.State current = lifeCycle.getCurrentState();
            if (current.isClosingOrClosed()) {
              throw new AlreadyClosedException(name + " is already " + current);
            }
            lifeCycle.startAndTransition(
                () -> proxy = createProxy.apply(peer), IOException.class);
          }
        }
      }
      return proxy;
    }

    Optional<PROXY> setNullProxyAndClose() {
      final PROXY p;
      synchronized (this) {
        p = proxy;
        lifeCycle.checkStateAndClose(() -> proxy = null);
      }
      return Optional.ofNullable(p);
    }

```

回到PeerProxyMap，核心的实例变量有两个

* Map<RaftPeerId, PeerAndProxy> peers
* CheckedFunction<RaftPeer, PROXY, IOException> createProxy

需要琢磨的方法为computeIfAbsent(RaftPeer)方法，该方法使用的是map.computeIfAbsent()方法，实参的函数式接口直接调用的PeerAndProxy类的构造器，再次强调，在PeerAndProxy的构造器中并没有对RaftClientProtocolClient实例变量进行赋值，而是在PeerAndProxy的getProxy()方法中使用单例模式动态赋值的，其需要使用外部类--PeerProxyMap的createProxy实例变量，其类型为函数式接口

```java
  public CheckedSupplier<PROXY, IOException> computeIfAbsent(RaftPeer peer) {
    final PeerAndProxy peerAndProxy = peers.computeIfAbsent(peer.getId(), k -> new PeerAndProxy(peer));
    return peerAndProxy::getProxy;
  }
```

PeerProxyMap实现了RaftPeer.Add接口，其addRaftPeers(Collection\<RaftPeer>)方法直接调用了computeIfAbsent(RaftPeer)方法

```java
  @Override
  public void addRaftPeers(Collection<RaftPeer> newPeers) {
    for(RaftPeer p : newPeers) {
      computeIfAbsent(p);
    }
  }
```

## 5. RaftClientRpcWithProxy

抽象类，持有一个PeerProxyMap\<Proxy>对象

```java
private final PeerProxyMap<PROXY> proxies;

protected RaftClientRpcWithProxy(PeerProxyMap<PROXY> proxies) {
    this.proxies = proxies;
}

  public PeerProxyMap<PROXY> getProxies() {
    return proxies;
  }
```

## 6. RaftClientRpc

最后的最后，是我们要看的Rpc类，是RaftClientRpcWithProxy的子类，持有两个实例变量

* ClientId clientId
* int maxMessageSize

首先要理解的是其构造器，构造器调用了父类--即RaftClientRpcWithProxy的构造器，RaftClientRpcWithProxy的构造器中需要传入一个PeerProxyMap对象，GrpcClientRpc直接调用了PeerProxyMap的构造器。而PeerProxyMap的构造器中核心的实参为createProxy函数式接口，即构造GrpcClientProtocolClient对象，这里传入的lambda表达式直接调用的为GrpcClientProtocolClient的构造器，根据参入的实参p(RaftPeer)构造不同的实例对象

```java
  public GrpcClientRpc(ClientId clientId, RaftProperties properties,
      GrpcTlsConfig adminTlsConfig, GrpcTlsConfig clientTlsConfig) {
    super(new PeerProxyMap<>(clientId.toString(),
        p -> new GrpcClientProtocolClient(clientId, p, properties, adminTlsConfig, clientTlsConfig)));
    this.clientId = clientId;
    this.maxMessageSize = GrpcConfigKeys.messageSizeMax(properties, LOG::debug).getSizeInt();
  }
```

接下来我们深入探索sendRequestAsync(RaftClientRequest)方法，探索整个RPC的流程

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
```

上述代码中，getProxies()返回一个PeerProxyMap对象，该对象封装了一个map，key为RaftPeerId，value为PeerProxyMap，调用器getProxy(RaftPeerId)方法，返回GrpcClientProtocolClient对象，如果PeerProxyMap中没有对应的GrpcClientProtocolClient对象，那么调用PeerProxyMap中的createProxy函数式接口创建GrpcClientProtocolClient对象，在GrpcClientRpc的构造器中，调用super()传递的函数式接口为GrpcClientProtocolClient的构造器。至此，返回了一个GrpcClientProtocolClient对象

接下来，通过GrpcClientProtocolClient.getOrderedStreamObservers()方法，返回一个AsyncStreamObservers对象，该对象持有一个StreamObserver\<RaftClientReplyProto>对象，作为输入流的回调接口，持有一个RequestStreamer对象，作为输出流的回调接口。

GrpcClientProtocolClient.getOrderedStreamObservers()定义如下

```java
  AsyncStreamObservers getOrderedStreamObservers() {
    return orderedStreamObservers.updateAndGet(
        a -> a != null? a : new AsyncStreamObservers(this::ordered));
  }
```

而ordered()方法定义如下

```java
  StreamObserver<RaftClientRequestProto> ordered(StreamObserver<RaftClientReplyProto> responseHandler) {
    return asyncStub.ordered(responseHandler);
  }
```

在这里，ordered()方法发起了RPC请求，该RPC请求属于Bidirectional类型，实参responseHandler为输入流的回调函数，返回值为输出流的回调函数。结合AsyncStreamObservers的构造器

```java
    AsyncStreamObservers(Function<StreamObserver<RaftClientReplyProto>, StreamObserver<RaftClientRequestProto>> f) {
      this.requestStreamer = new RequestStreamer(f.apply(replyStreamObserver));
    }
```

这里在构造RequestStreamer对象时，已经调用了f.apply()方法，即GrpcClientProtocolClient.ordered()方法，那么已经发起了RPC请求，体哦概念股的输入流回调函数为AsyncStreamObservers中的实例变量replyStreamObserver，返回值为输出流的回调函数，将其作为实参封装在RequestStreamer实例中，接下来的onNext(RaftClientRequest)方法，实际上调用的为AsyncStreamObservers.onNext()方法，AsyncStreamObservers.onNext()方法实际上是对RequestStreamer.onNext()方法的封装，而RequestStreamer.onNext()方法实际上是对输出流StreamOberServer\<RaftClientRequest>.onNext()方法的封装。提到此，需要回到GrpcClientProtocolClient类中的看一下getOrderedStreamObservers()方法和getUnorderedAsyncStreamObservers()方法的异同了。

回到GrpcClientRpc中，看接下来的sendRequestAsyncUnordered(RaftClientRequest)方法，逻辑上同sendRequestAsync(RaftClientRequest)方法一致，只是底层RPC的不同。Unordered()方法使用的是unordered RPC，另外一个使用的是ordered RPC。

```java
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

接下来看sendRequest(RaftClientRequest)方法和sendRequest(RaftClientRequest, GrpcClientProtocolClient)方法