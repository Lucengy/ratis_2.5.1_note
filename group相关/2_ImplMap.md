## 1. 前言

RaftServerProxy的内部类，顾名思义，是一个map

```
RaftGroupId --> RaftServerImpl
```

这意味着，每个RaftServer实例上，可以有国歌RaftServerImpl对象，而每一个RaftServerImpl对象，都有唯一的一个RaftGroupId。

## 2. 实现

ImplMap持有一个map，该map默认会被初始化

```java
private final ConcurrentMap<RaftGroupId, CompletableFuture<RaftServerImpl> map = 
    new ConcurrentHashMap<>();
```

addNew()方法加入了很多判断和断言，其核心逻辑就是一个简单的map.put()操作。这里使用的是newRaftServerImpl方法构造的新的RaftServerImpl对象

```java
synchronized CompletableFuture<RaftServerImpl> addNew(RaftGroup group, StartupOption option) {
    if (isClosed) {
        return JavaUtils.completeExceptionally(new AlreadyClosedException(
            getId() + ": Failed to add " + group + " since the server is already closed"));
    }
    if (containsGroup(group.getGroupId())) {
        return JavaUtils.completeExceptionally(new AlreadyExistsException(
            getId() + ": Failed to add " + group + " since the group already exists in the map."));
    }
    final RaftGroupId groupId = group.getGroupId();
    final CompletableFuture<RaftServerImpl> newImpl = newRaftServerImpl(group, option);
    final CompletableFuture<RaftServerImpl> previous = map.put(groupId, newImpl);
    Preconditions.assertNull(previous, "previous");
    LOG.info("{}: addNew {} returns {}", getId(), group, toString(groupId, newImpl));
    return newImpl;
}
```

