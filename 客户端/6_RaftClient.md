## 1. 前言

看完RaftClientRequest，再看RaftClient类，是怎么使用RaftClientRequest的。

先看RaftClientRpc类，这是client侧的rpc类，可以看到其send方法都是以RaftClientRequest对象作为形参的。

服务端，直接了当的来看StateMachine类，可以看到其startTransaction()方法，形参类型为RaftCilentRequest对象。

RaftClient是一个接口，用来定义各种各样的api。持有一个Builder内部类，build()方法用来构造RaftClient对象。

但是这里并没有找到RaftClientRequest的相关内容，先记录API

```java
public interface RaftClient extends Closeable {
  Logger LOG = LoggerFactory.getLogger(RaftClient.class);

  /** @return the id of this client. */
  ClientId getId();

  /** @return the group id of this client. */
  RaftGroupId getGroupId();

  /** @return the cluster leaderId recorded by this client. */
  RaftPeerId getLeaderId();

  /** @return the {@link RaftClientRpc}. */
  RaftClientRpc getClientRpc();

  /** @return the {@link AdminApi}. */
  AdminApi admin();

  /** Get the {@link GroupManagementApi} for the given server. */
  GroupManagementApi getGroupManagementApi(RaftPeerId server);

  /** Get the {@link SnapshotManagementApi} for the given server. */
  SnapshotManagementApi getSnapshotManagementApi();

  /** Get the {@link SnapshotManagementApi} for the given server. */
  SnapshotManagementApi getSnapshotManagementApi(RaftPeerId server);

  /** Get the {@link LeaderElectionManagementApi} for the given server. */
  LeaderElectionManagementApi getLeaderElectionManagementApi(RaftPeerId server);

  /** @return the {@link BlockingApi}. */
  BlockingApi io();

  /** Get the {@link AsyncApi}. */
  AsyncApi async();

  /** @return the {@link MessageStreamApi}. */
  MessageStreamApi getMessageStreamApi();

  /** @return the {@link DataStreamApi}. */
  DataStreamApi getDataStreamApi();
}
```

## 2. RaftClientImpl

### 1. PendingClientRequest

静态内部抽象类

### 2. RaftPeerList

静态内部类，用来存在当前raftGroup中的RaftPeer列表

只有遍历和批量替换这两种方法

```java
static class RaftPeerList implements Iterable<RaftPeer> {
    private final AtomicReference<List<RaftPeer>> list = new AtomicReference<>();
    
    @Override
    public Iterator<RaftPeer> iterator() {
        return list.get().iterator();
    }
    
    void set(Collection<RaftPeer> newPeers) {
        Preconditions.assertTrue(!newPeers.isEmpty());
        list.set(Collections.unmodifiableList(new ArrayList<>(newPeers)));
    }
}
```

