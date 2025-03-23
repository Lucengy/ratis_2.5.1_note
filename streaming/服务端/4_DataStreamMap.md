## 1. 前言

接口，持有一个map，ClientInvocationId --> DataStream

提供了两个Api方法，分别为

* computeIfAbsent
* remove

```java
public interface DataStreamMap {
    CompletableFuture<DataStream> computeIfAbsent(ClientInvocationId invocationId,
         Function<ClientInvocationId, CompletableFuture<DataStream>> newDataStream);

    /** Similar to {@link java.util.Map#remove(java.lang.Object). */
    CompletableFuture<DataStream> remove(ClientInvocationId invocationId);
}
```

## 2. 实现类DataStreamMapImpl

说白了，具体CompletableFuture\<DataStream>的获取，是根据computeIfAbsent的实参来得到的

```java
class DataStreamMapImpl implements DataStreamMap {
    public static final Logger LOG = LoggerFactory.getLogger(DataStreamMapImpl.class);

    private final String name;
    private final ConcurrentMap<ClientInvocationId, CompletableFuture<DataStream>> map = new ConcurrentHashMap<>();

    DataStreamMapImpl(Object name) {
        this.name = name + "-" + JavaUtils.getClassSimpleName(getClass());
    }

    @Override
    public CompletableFuture<DataStream> remove(ClientInvocationId invocationId) {
        return map.remove(invocationId);
    }

    @Override
    public CompletableFuture<DataStream> computeIfAbsent(ClientInvocationId invocationId,
            Function<ClientInvocationId, CompletableFuture<DataStream>> newDataStream) {
        return map.computeIfAbsent(invocationId, newDataStream);
    }

    @Override
    public String toString() {
        return name;
    }
}
```

