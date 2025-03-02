## 1. 前言

该类为PeerProxyMap类，根据JavaDoc

A map from peer id to peer and its proxy

## 2. 先看其内部类，PeerAndProxy

PROXY泛型为其外部类定义的泛型，对其的要求就是extends Closeable。这里的意思就是一个peer和它对应的代理对象，这个代理对象可以是任何类型，这里做了极大的抽象，做成了通用类

```java
private class PeerAndProxy {
    private final RaftPeer peer;
    private volatile PROXY proxy;
    private final LifeCycle lifeCycle;
    
    PeerAndProxy(RaftPeer peer) {
        this.peer = peer;
        this.lifeCycle = new LifeCycle(peer);
    }
    
    RaftPeer getPeer() {
        return peer;
    }
    
    PROXY getProxy() throws IOException {
        if(proxy == null) {
            synchronized(this) {
                if(proxy == null) {
                    final LifeCycle.state current = lifeCycle.getCurrentState();
                    if(current.isClosingOrClosed()) {
                        throw new AlreadClosedException();
                    }
                    
                    lifeCycle.startAndTransition(() -> proxy = createProxy.apply(peer), IOException.class);
                }
            }
        }
        
        return proxy;
    }
    
    Optional<PROXY> setNullProxyAndClose() {
        final PROXY p;
        synchronized(this) {
            p = proxy;
            lifeCycle.checkStateAndClose(() -> proxy = null);
        }
        
        return Optional.ofNullable(p);
    }
}
```

## 3. PeerProxyMap

```java
public class PeerProxyMap<PROXY extends Closeable> implements RaftPeer.Add, Closeable {
    private final String name;
    
    private final Map<RaftPeerId, PeerAndProxy> peers = new ConcurrentHashMap<>();
    
    private final Object resetLock = new Object();
    
    private final CheckedFunction<RaftPeer, PROXY, IOException> createProxy;
    
    public PeerProxyMap(String name, CheckedFunction<RaftPeer, PROXY, IOException> createProxy) {
        this.name = name;
        this.createProxy = createProxy;
    }
    
    public PeerProxyMap(String name) {
        this.name = name;
        this.createProxy = this::createProxyImpl;
    }
    
    public PROXY createProxyImpl(RaftPeer peer) throws IOException {
        throw new UnsupportedOperationException();
    }
    
    public PROXY getProxy(RaftPeerId id) throws IOException {
        Objects.requireNonNull(id, "id == null");
        PeerAndProxy p = peers.get(id);
        if (p == null) {
            synchronized(resetLock) {
                p = Objects.requireNonNull(peers.get(id));
            }
        }
        
        return p.getProxy();
    }
    
    public void addRaftPeers(Collection<RaftPeer> newPeers) {
        for(RaftPeer p : newPeers) {
            computeIfAbsent(p);
        }
    }
    
    public CheckedSupplier<PROXY, IOException> computeIfAbsent(RaftPeer peer) {
        final PeerAndProxy peerAndProxy = peers.computeIfAbsent(peer.getId(), k -> new PeerAndProxy(peer));
        return peerAndProxy::getProxy;
    }
    
    public void resetProxy(RaftPeerId id) {
        final PeerAndProxy pp;
        Optional<PROXY> optional = Optional.empty();
        synchronized(resetLock) {
            pp = peers.remove(id);
            if(pp != null) {
                final RaftPeer peer = pp.getPeer();
                optional = pp.setNullProxyAndClose();
                computeIfAbsent(peer);
            }
        }
        
        optional.ifPresent(proxy -> closeProxy(proxy, pp));
    }
    
    public boolean handleException(RaftPeerId serverId, Throwable e, boolean reconnect) {
        if (reconnect || IOUtils.shouldReconnect(e)) {
          resetProxy(serverId);
          return true;
        }
        return false;
  	}
}
```

