## 1. 前言

这部分留给multi Raft

## 2. 内部类ImplMap

持有一个map，key为RaftGroupId，value为CompletableFuture\<RaftServerImpl>

```java
private final ConcurrentMap<RaftGroupId, CompletableFuture<RaftServerImpl>> map = new ConcurrentHashMap<>();
```

这里需要关注的一个方法是addNew(raftGroup group)方法，因为整体代码比较简单，就全贴过来，这里的newRaftServerImpl(RaftGroup)方法属于RaftServerProxy

```java
    synchronized CompletableFuture<RaftServerImpl> addNew(RaftGroup group) {
      if (isClosed) {
        return JavaUtils.completeExceptionally(new AlreadyClosedException(
            getId() + ": Failed to add " + group + " since the server is already closed"));
      }
      if (containsGroup(group.getGroupId())) {
        return JavaUtils.completeExceptionally(new AlreadyExistsException(
            getId() + ": Failed to add " + group + " since the group already exists in the map."));
      }
      final RaftGroupId groupId = group.getGroupId();
      final CompletableFuture<RaftServerImpl> newImpl = newRaftServerImpl(group);
      final CompletableFuture<RaftServerImpl> previous = map.put(groupId, newImpl);
      Preconditions.assertNull(previous, "previous");
      LOG.info("{}: addNew {} returns {}", getId(), group, toString(groupId, newImpl));
      return newImpl;
    }
```

## 3

