## Ratis-234

```
When a client request is specified with ALL replication, it is possible that it is committed (i.e. replicated to a majority of servers) but not yet replicated to all servers. This feature is to let the client to watch it until it is replicated to all server.
```

client可以指定replication必须发送到所有的replication中，而不是majority/quorum。

在Raft.proto文件中可以看到

```protobuf
enum ReplicationLevel {
  /** Committed at the leader and replicated to the majority of peers. */
  MAJORITY = 0;
  /** Committed at the leader and replicated to all peers.
       Note that ReplicationLevel.ALL implies ReplicationLevel.MAJORITY. */
  ALL = 1;

  /** Committed at majority peers.
      Note that ReplicationLevel.MAJORITY_COMMITTED implies ReplicationLevel.MAJORITY. */
  MAJORITY_COMMITTED = 2;

  /** Committed at all peers.
      Note that ReplicationLevel.ALL_COMMITTED implies ReplicationLevel.ALL
      and ReplicationLevel.MAJORITY_COMMITTED */
  ALL_COMMITTED = 3;
}
```

## Ratis270 是retry cache相关的，目前还没看

## Ratis345 Watch requests should bypass sliding window

```
Watch requests are special read-only requests. Once the watch condition is satisfied, it should be returned immediately. It should not wait for the earlier requests, if there are any.

Currently, the watch requests are put in the sliding window so that it won't return until all earlier requests have been returned.
```

这里说明的应该是server端的SlidingWindow的处理