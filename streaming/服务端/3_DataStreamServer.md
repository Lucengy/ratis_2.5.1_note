## 1. API

```java
public interface DataStreamServer extends Closeable {
    DataStreamServerRpc getServerRpc();
}
```

## 2. 实现类DataStreamServerImpl

```java
class DataStreamServerImpl implements DataStreamServer {
    private final DataStreamServerRpc serverRpc;
    
    DataStreamServerImpl(RaftServer server, Parameters parameters) {
        final SupportedDataStreamType type = RaftConfigKeys.DataStream.type(server.getProperties(), LOG::info);
        this.serverRpc = DataStreamServerFactory.newInstance(type, parameters).newDataStreamServerRpc(server);
    }
    
    @Override
    public DataStreamServerRpc getServerRpc() {
        return serverRpc;
    }

    @Override
    public void close() throws IOException {
        serverRpc.close();
    }
}
```

