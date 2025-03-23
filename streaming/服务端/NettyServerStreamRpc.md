## 1. 前言

ratis-streaming的客户端rpc类为DataStreamClientRpc，对应实现类为NettyClientStreamRpc，与之对应的：服务端rpc类为DataStreamServerRpc，对应实现类为NettyServerStreamRpc

在RaftServerProxy.startImpl()方法中，调用了getDataStreamServerRpc().start()方法，即DataStreamServerRpc.start()方法。DataStreamServerRpc实例对象通过DataStreamServerImpl进行构造，在RaftServerProxy的构造器中

```java
this.dataStreamServerRpc = new DataStreamServerImpl(this, parameters).getServerRpc();
```



## 2. 接口DataStreamServerRpc

```java
public interface DataStreamServerRpc extends RaftPeer.Add, Closeable {
    void start();
    InetSocketAddress getInetSocketAddress();
}
```

## 3. 实现类NettyServerStreamRpc

### 1. 静态内部类Proxies

```java
/** Proxies to other peers. */
```

乍一看这里是primary存放standby  rpc stub的地方。持有发一个PeerProxyMap实例变量，该实例变量是一个map，里面存放的是PeerAndProxy的一个map。因为NettyServerStreamRpc也是RaftServerProxy层面的，那么实际上一台server上只有一个rpc入口，根据groupID再分发到对应的RaftGroup，即Raft ServerImpl中。

```java
static class Proxies {
    private final PeerProxyMap<DataStreamClient> map;
    
    Proxies(PeerProxyMap<DataStreamClient> map) {
        this.map = map;
    }
    
    void addPeers(Collection<RaftPeer> newPeers) {
        map.addRaftPeers(newPeers);
    }
    
    void close() {
        map.close();
    }
    
    Set<DataStreamOutputRpc> getDataStreamOutput(RaftClientRequest request, Set<RaftPeer> peers) throws IOException{
        final Set<DataStreamOutputRpc> outs = new HashSet<>();
        try{
            getDataStreamOutput(reqeust, peers, outs);
        } catch(IOException e) {
            outs.forEach(DataStreamOutputRpc::closeAsync);
            throw e;
        }
        
        return outs;
    }
    
    private void getDataStreamOutput(RaftClientRequest request, Set<RaftPeer> peers, Set<DataStreamOutputRpc> outs) throws IOException{
        for(RaftPeer peer : peers) {
            outs.add((DataStreamOutputRpc)map.computeIfAbsent(peer).get().stream(request));
        } catch(IOExcepiton e) {
            map.handleException(peer.getId(), e, true);
            throw new IOException();
        }
    }
}
```

### 2. 静态内部类ProxiesPool

原则上来说，只有一个Proxies已经够用了，这里为什么还有一个ProxiesPool，真的是让人捉摸不透，根据RATIS-1371，

```
If server1 use only one client to transfer data to server2, there is bottleneck, so we should use multi-client.
```

也就是说，raftServer到raftServer之间存在着多条rpc链路，clientPoolSize默认为10。这里的get方法，通过clientId和streamId做hash，确保了Proxies的唯一性

```java
static class ProxiesPool {
    private final List<Proxies> list;
    
    ProxiesPool(String name, RaftProperties properties) {
        final int clientPoolSize = RaftServerConfigKeys.DataStream.clientPoolSize(properties);
        final List<Proxies> proxies = new ArrayList<>(clientPoolSize);
        for(int i = 0; i < clientPoolSize; i ++) {
            proxies.add(new Proxies(new PeerProxyMap<>(name, peer -> newClient(peer, properties))));
        }
        this.list = Collections.unmodifiableList(proxies);
    }
    
    void addRaftPeers(Collection<RaftPeer> newPeers) {
        list.forEach(proxy -> proxy.addPeers(newPeers));
    }
    
    Proxies get(DataStreamPacket p) {
        final long hash = Integer.toUnsignedLong(Objects.hash(p.getClientId(), p.getStreamId()));
        return list.get(Math.toIntExact(hash % list.size()));
    }
    
    void close() {
        list.forEach(Proxies::close);
    }
}
```

### 3. 静态 内部类RequestRef

封装了对DataStreamRequestByteBuf的引用

```java
static class ReqeustRef {
    private final AtomicReference<DataStreamRequestByteBuf> ref = new AtomicReference<>();
    
    UnchekcedAutoClsoeable set(DataStreamRequestByteBuf current) {
        final DataStreamRequestByteBuf previous = ref.getAndUpdate(p -> p == null? current : p);
        Preconditions.assertNull(previous);
    }
    
    DataStreamRequestByteBuf getAndSetNull() {
        return ref.getAndSet(null);
    }
}
```

