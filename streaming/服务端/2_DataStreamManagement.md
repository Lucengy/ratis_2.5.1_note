## 1. 前言

先看composeAsync方法

```java
static <T> CompletableFuture<T> composeAsync(AtomicReference<CompletableFuture<T>> future, Executor executor, Function<T, CompletableFuture<T> function) {
    return future.getAndGet(previous -> previous.thenComposeAsync(function, execute));
}
```

## 2. 静态内部类LocalStream

```java
static class LocalStream {
    private final CompletableFuture<DataStream> streamFuture;
    private final AtomicReference<CompletableFuture<Long>> writeFuture;
    
    LocalStream(CompletableFuture<DataStream> streamFuture) {
        this.streamFuture = streamFuture;
        this.writeFuture = new AtomicReference<>(streamFuture.thenApply(s -> 0L));
    }
    
    CompletableFuture<Long> write(ByteBuf buf, WriteOption[] options, Executor executor) {
        return composeAsync(writeFuture, executor, n -> streamFuture.thenCompose(
         	stream -> writeToAsync(buf, options, stream, executor)
        	)
       );
    }
}
```

接下来看writeToAsync()方法

```java
static CompletableFuture<Long> writeToAsync(ByteBuf buf, WriteOption[] options, DataStream stream,
      Executor defaultExecutor) {
    final Executor e = Optional.ofNullable(stream.getExecutor()).orElse(defaultExecutor);
    return CompletableFuture.supplyAsync(() -> writeTo(buf, options, stream), e);
}
```

这里调用的是writeTo方法

```java
static long writeTo(ByteBuf buf, WriteOption[] options, DataStream stream) {
    final DataChannel channel = stream.getDataChannel();
    long byteWritten = 0;
    for (ByteBuffer buffer : buf.nioBuffers()) {
        final ReferenceCountedObject<ByteBuffer> wrapped = ReferenceCountedObject.wrap(buffer, buf::retain, buf::release);
        try {
            byteWritten += channel.write(wrapped);
        } catch (Throwable t) {
            throw new CompletionException(t);
        }
    }

    if (WriteOption.containsOption(options, StandardWriteOption.SYNC)) {
        try {
            channel.force(false);
        } catch (IOException e) {
            throw new CompletionException(e);
        }
    }

    if (WriteOption.containsOption(options, StandardWriteOption.CLOSE)) {
        close(stream);
    }
    return byteWritten;
}
```

## 3. 静态内部类RemoteStream

这里的out实例变量对应着pipeline的下游节点的输出流，一个RemoteStream变量持有一个输出流对象，符合对pipeline的常规认知

```java
static class RemoteStream {
    private final DataStreamOutputRpc out;
    private final AtomicReference<CompletableFuture<DataStreamReply>> sendFuture
        = new AtomicReference<>(CompletableFuture.completedFuture(null));
    private final RequestMetrics metrics;

    RemoteStream(DataStreamOutputRpc out, RequestMetrics metrics) {
        this.metrics = metrics;
        this.out = out;
    }

    CompletableFuture<DataStreamReply> write(DataStreamRequestByteBuf request, Executor executor) {
        final RequestContext context = metrics.start();
        return composeAsync(sendFuture, executor,
                            n -> out.writeAsync(request.slice().nioBuffer(), request.getWriteOptions())
                            .whenComplete((l, e) -> metrics.stop(context, e == null)));
    }
}
```

## 4. 静态内部类StreamInfo

StreamInfo让人摸不到头脑，主要是因为这里封装了一个RemoteStream的set。在常规pipeline中，一个RaftServerImpl应该持有两个输出流：一个为LocalStream，负责写入本地；一个RemoteStream，负责写入下游pipeline。但是这里却用了set。

果然中国人的代码就是这么难懂。这是因为ratis-streaming在设计之初，为了简单，先使用的是star topology，所以收到client请求的第一个节点作为primary节点，将request发给剩余raftPeer节点，如此便定下了StreamInfo中持有一个RemoteStream set的基调，但是在RATIS-1248中，进行了拓展

```
Not only support primary -> peer1 && peer2, but also support primary -> peer1 -> peer2.
```

首先，在RaftRpcRequestProto中加入了RoutingTable信息

```protobuf
message RaftRpcRequestProto {
  bytes requestorId = 1;
  bytes replyId = 2;
  RaftGroupIdProto raftGroupId = 3;
  uint64 callId = 4;
  bool toLeader = 5;

  uint64 timeoutMs = 13;
  RoutingTableProto routingTable = 14;
  SlidingWindowEntry slidingWindowEntry = 15;
}
```

有关RouteingTableProto的定义就颇为有趣，一个RoutingTable持有多个RouteEntry，每个RouteEntry又是如何定义的呢？当前节点，即peerId，以及它的后继节点们，即(repeted)successors。

这里，后继节点还是集合，但是啊，但是，可但是，但可是，client侧在构造pipeline时，所有节点只有一个后继节点。没错，**只有一个元素的集合也TM是集合**，真的是秀儿本秀

```protobuf
message RouteProto {
  RaftPeerIdProto peerId = 1;
  repeated RaftPeerIdProto successors = 2;
}

message RoutingTableProto {
  repeated RouteProto routes = 1;
}
```

其调用方，以FileStore和Ozone中的代码为例

1. FileStore中，SubCommandBase.java

   ```java
   public RoutingTable getRoutingTable(Collection<RaftPeer> raftPeers, RaftPeer primary) {
       RoutingTable.Builder builder = RoutingTable.newBuilder();
       RaftPeer previous = primary;
       for (RaftPeer peer : raftPeers) {
         if (peer.equals(primary)) {
           continue;
         }
         builder.addSuccessor(previous.getId(), peer.getId());
         previous = peer;
       }
   
       return builder.build();
     }
   ```

2. Ozone中，RatisHelper.java

   ```java
   public static RoutingTable getRoutingTable(Pipeline pipeline) {
       RaftPeerId primaryId = null;
       List<RaftPeerId> raftPeers = new ArrayList<>();
   
       for (DatanodeDetails dn : pipeline.getNodesInOrder()) {
         final RaftPeerId raftPeerId = RaftPeerId.valueOf(dn.getUuidString());
         try {
           if (dn == pipeline.getClosestNode()) {
             primaryId = raftPeerId;
           }
         } catch (IOException e) {
           LOG.error("Can not get primary node from the pipeline: {} with " +
               "exception: {}", pipeline, e.getLocalizedMessage());
           return null;
         }
         raftPeers.add(raftPeerId);
       }
   
       RoutingTable.Builder builder = RoutingTable.newBuilder();
       RaftPeerId previousId = primaryId;
       for (RaftPeerId peerId : raftPeers) {
         if (peerId.equals(primaryId)) {
           continue;
         }
         builder.addSuccessor(previousId, peerId);
         previousId = peerId;
       }
   
       return builder.build();
     }
   ```

   

```java
static class StreamInfo {
    private final RaftClientRequest request;
    private final boolean primary;
    private final LocalStream local;
    private final Set<RemoteStream> remotes;
    private final RaftServer server;
    
    private final AtomicReference<CompletableFuture<Void>> previous = 
        new AtomicReference<>(CompletableFuture.completedFuture(null));
    
    StreamInfo(RaftClientRequest request, boolean primary, CompletableFuture<DataStream> stream RaftServer server, CheckedBiFunction<RaftClientRequest, Set<RaftPeer>, Set<DataStreamOutputRpc>, IOException> getStreams, Function<ReqeustType, RequestMetrics> metricConstructor) throws IOException{
        this.request = request;
        this.primary = primary;
        this.local = new LocalStream(stream, metricsConstructor.apply(RequestType.LOCAL_WRITE));
        this.server = server;
        final Set<RaftPeer> successors = getSuccessors(server.getId());
        final Set<DataStreamOutputRpc> outs = getStreams.apply(request, successors);
        this.remotes = outs.stream()
            .map(o -> new RemoteStream(o, metricsConstructor.apply(RequestType.REMOTE_WRITE)))
            .collect(Collectors.toSet());
    }
    
    AtomicReference<CompletableFuture<Void>> getPrevious() {
        return previous;
    }

    RaftClientRequest getRequest() {
        return request;
    }

    Division getDivision() throws IOException {
        return server.getDivision(request.getRaftGroupId());
    }

    Collection<CommitInfoProto> getCommitInfos() {
        try {
            return getDivision().getCommitInfos();
        } catch (IOException e) {
            throw new IllegalStateException(e);
        }
    }

    boolean isPrimary() {
        return primary;
    }

    LocalStream getLocal() {
        return local;
    }
    
    <T> List<T> applyToRemotes(Function<RemoteStream, T> function) {
        return remotes.isEmpty()?Collections.emptyList(): 				remotes.stream().map(function).collect(Collectors.toList());
    }
    
    private Set<RaftPeer> getSuccessors(RaftPeerId peerId) throws IOException {
        final RaftConfiguration conf = getDivision().getRaftConf();
        final RoutingTable routingTable = request.getRoutingTable();

        if (routingTable != null) {
            return routingTable.getSuccessors(peerId).stream().map(conf::getPeer).collect(Collectors.toSet());
        }

        if (isPrimary()) {
            // Default start topology
            // get the other peers from the current configuration
            return conf.getCurrentPeers().stream()
                .filter(p -> !p.getId().equals(server.getId()))
                .collect(Collectors.toSet());
        }

        return Collections.emptySet();
    }
    
}
```

## 5. 静态内部类StreamMap

```java
static class StreamMap {
    private final ConcurrentMap<ClientInvocationId, StreamInfo> map = new ConcurrentHashMap<>();

    StreamInfo computeIfAbsent(ClientInvocationId key, Function<ClientInvocationId, StreamInfo> function) {
        final StreamInfo info = map.computeIfAbsent(key, function);
        LOG.debug("computeIfAbsent({}) returns {}", key, info);
        return info;
    }

    StreamInfo get(ClientInvocationId key) {
        final StreamInfo info = map.get(key);
        LOG.debug("get({}) returns {}", key, info);
        return info;
    }

    StreamInfo remove(ClientInvocationId key) {
        final StreamInfo info = map.remove(key);
        LOG.debug("remove({}) returns {}", key, info);
        return info;
    }
}
```

