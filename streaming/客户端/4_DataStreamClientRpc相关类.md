## 1. 前言

Client侧的RPC接口定义，用来发送数据，目前只有一个实现类NettyClientStreamRpc

## 2. 接口定义

```java
public interface DataStreamClientRpc extends Closeable {
  /** Async call to send a request. */
  default CompletableFuture<DataStreamReply> streamAsync(DataStreamRequest request) {
    throw new UnsupportedOperationException(getClass() + " does not support "
        + JavaUtils.getCurrentStackTraceElement().getMethodName());
  }
}
```

## 3. NettyClientStreamRpc实现类

发送逻辑在streamAsync(DataStreamRequest)方法中；输入流的回调函数正在getClientHandler()方法中，用来监控服务端发回的ack

### 1. WorkerGroupGetter

实现了Supplier\<EventLoopGroup>，就是提供了EventLoopGroup get()方法。同时根据变量名，这里的EventLoopGroup是一个共用的

```java
private static final AtomicReference<EventLoopGroup> SHARED_WORKER_GROUP = new AtomicReference<>();
```

在构造器中，根据需要对SHARED_WORKER_GROUP进行初始化。

```java
private static class WorkerGroupGetter implements Supplier<EventLoopGroup> {
    private static final AtomicReference<EventLoopGroup> SHARED_WORKER_GROUP = new AtomicReference<>();

    static EventLoopGroup newWorkerGroup(RaftProperties properties) {
        return NettyUtils.newEventLoopGroup(
            JavaUtils.getClassSimpleName(NettyClientStreamRpc.class) + "-workerGroup",
            NettyConfigKeys.DataStream.Client.workerGroupSize(properties), false);
    }

    private final EventLoopGroup workerGroup;
    private final boolean ignoreShutdown;

    WorkerGroupGetter(RaftProperties properties) {
        if (NettyConfigKeys.DataStream.Client.workerGroupShare(properties)) {
            workerGroup = SHARED_WORKER_GROUP.updateAndGet(g -> g != null? g: newWorkerGroup(properties));
            ignoreShutdown = true;
        } else {
            workerGroup = newWorkerGroup(properties);
            ignoreShutdown = false;
        }
    }

    @Override
    public EventLoopGroup get() {
        return workerGroup;
    }

    void shutdownGracefully() {
        if (!ignoreShutdown) {
            workerGroup.shutdownGracefully();
        }
    }
}
```



### 2.内部类Connection

这里所有的实例变量都使用final进行修饰

```java
static class Connection {
    private final InetSocketAddress address;
    private final WorkerGroupGetter workerGroup;
    private final Supplier<ChannelInitializer<SocketChannel>> channelInitializerSupplier;
    private final AtomicReference<ChannelFuture> ref;
}
```

构造器也是对这四个实例变量进行赋值

```java
Connection(InetSocketAddress address, WorkerGroupGetter workerGroup,
        Supplier<ChannelInitializer<SocketChannel>> channelInitializerSupplier) {
      this.address = address;
      this.workerGroup = workerGroup;
      this.channelInitializerSupplier = channelInitializerSupplier;
      this.ref = new AtomicReference<>(connect());
}
```

首先看connect()方法，这是在构造器中对AtomicReference\<ChannelFuture>赋值时调用的。就是Netty的一个demo调用，这里注意的点就是handler()函数为实例对象时的实参，所有输入输出流的逻辑都在该对象中

```java
private ChannelFuture connect() {
    return new Bootstrap()
        .group(getWorkerGroup())
        .channel(NioSocketChannel.class)
        .handler(channelInitializerSupplier.get())
        .option(ChannelOption.SO_KEEPALIVE, true)
        .connect(address)
        .addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) {
                if (!future.isSuccess()) {
                    scheduleReconnect(this + " failed", future.cause());
                } else {
                    LOG.trace("{} succeed.", this);
                }
            }
        });
}
```

剩下的几个方法没有什么逻辑含量，先跳过

### 3. ReplyQueue

该内部类持有一个Queue，element为CompletableFuture\<DataStreamReply>对象。即这是发送方用来存放已发送但是未获得ack的容器

```java
static class ReplyQueue implements Iterable<CompletableFuture<DataStreamReply>> {
    static final ReplyQueue EMPTY = new ReplyQueue();
    
    private final Queue<CompletableFuture<DataStreamReply>> queue = new ConcurrentLinkedQueue<>();
    
    private int emptyId; //没有进行赋值，默认为0咯
    
    synchronized Integer getEmptyId() {
        return queue.isEmpty()? emptyId: null;
    }
    
    synchronized boolean offer(CompletableFuture<DataStreamReply> f) {
        if(queue.offer(f)) {
            emptyId ++;
            return true;
        }

        return false;
    }
    
    CompletableFuture<DataStreamReply> poll() {
        return queue.poll();
    }
    
    int size() {
        return queue.size();
    }
    
    @Override
    public Iterable<CompletableFuture<DataStreamReply>> interator() {
        return queue.iterator();
    }
}
```

### 4. NettyClientStreamRpc

实例变量比较简单，只有三个，均使用final进行修饰，且replies还做了默认初始化

```java
private final String name;
private final Connection connection;
private final ConcurrentMap<ClientInvocationId, ReplyQueue> replies = new ConcurrentHashMap<>();
```

那么构造器中需要注意的就是Connection实例对象connection的构造了，在上面说了，Connection在构造的时候会传入channelInitializerSupplier对象，所有输出流输出流的处理均在该对象中

```java
public NettyClientStreamRpc(RaftPeer server, TlsConf tlsConf, RaftProperties properties) {
    this.name = JavaUtils.getClassSimpleName(getClass()) + "->" + server;
    this.replyQueueGracePeriod = NettyConfigKeys.DataStream.Client.replyQueueGracePeriod(properties);

    final InetSocketAddress address = NetUtils.createSocketAddr(server.getDataStreamAddress());
    final SslContext sslContext = NettyUtils.buildSslContextForClient(tlsConf);
    this.connection = new Connection(address,
                                     new WorkerGroupGetter(properties),
                                     () -> newChannelInitializer(address, sslContext, 													getClientHandler()));
}
```

这里的Supplier\<ChannelInitializer\<SocketChannel>>对象使用lambda表达式构造，其get()方法为newChannnelInitializer()方法，该方法传入了5个handler，其中ssl可以先忽略。

前两个为编码相关，即输出流的处理逻辑

后两个为解码相关，这里的handler为getClientHandler()方法返回的，getClientHandler()方法返回ChannelInboundHandler对象。故

* 输出流的逻辑在newEncoder()和newEncoderDataStreamRequestFilePositionCount()方法中

* 输入流的逻辑在newDecoder()和getClientHandler()方法中

```java
static ChannelInitializer<SocketChannel> newChannelInitializer(
      InetSocketAddress address, SslContext sslContext, ChannelInboundHandler handler) {
    return new ChannelInitializer<SocketChannel>(){
        @Override
        public void initChannel(SocketChannel ch) {
            ChannelPipeline p = ch.pipeline();
            if (sslContext != null) {
                p.addLast("ssl", sslContext.newHandler(ch.alloc(), address.getHostName(), address.getPort()));
            }
            p.addLast(newEncoder());
            p.addLast(newEncoderDataStreamRequestFilePositionCount());
            p.addLast(newDecoder());
            p.addLast(handler);
        }
    };
}
```

紧接着就是其核心方法，即实现的DataStreamClientRpc中的streamAsync()方法

```java
@Override
public CompletableFuture<DataStreamReply> streamAsync(DataStreamRequest request) {
    final CompletableFuture<DataStreamReply> f = new CompletableFuture<>();
    ClientInvocationId clientInvocationId = ClientInvocationId.valueOf(request.getClientId(), request.getStreamId());
    final ReplyQueue q = replies.computeIfAbsent(clientInvocationId, key -> new ReplyQueue());
    if (!q.offer(f)) {
        f.completeExceptionally(new IllegalStateException(this + ": Failed to offer a future for " + request));
        return f;
    }
    final Channel channel = connection.getChannelUninterruptibly();
    if (channel == null) {
        f.completeExceptionally(new AlreadyClosedException(this + ": Failed to send " + request));
        return f;
    }
    LOG.debug("{}: write {}", this, request);
    channel.writeAndFlush(request).addListener(future -> {
        if (!future.isSuccess()) {
            final IOException e = new IOException(this + ": Failed to send " + request, future.cause());
            LOG.error("Channel write failed", e);
            f.completeExceptionally(e);
        }
    });
    return f;
}
```

