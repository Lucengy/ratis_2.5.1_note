## 1. 前言

在看Ozone的时候，突然意识到不清楚Ozone是怎么使用Ratis的，回到Ratis中，从CounterServer出发，看一下user application是怎么使用ratis的.

代码存在于CounterServer中，可以看到入口类是RaftServer，通过调用RaftServer.start()方法启动了服务类

```java
private final RaftServer server;

public CounterServer(RaftPeer peer, File storageDir) throws IOException {
    //create a property object
    final RaftProperties properties = new RaftProperties();

    //set the storage directory (different for each peer) in the RaftProperty object
    RaftServerConfigKeys.setStorageDir(properties, Collections.singletonList(storageDir));

    //set the port (different for each peer) in RaftProperty object
    final int port = NetUtils.createSocketAddr(peer.getAddress()).getPort();
    GrpcConfigKeys.Server.setPort(properties, port);

    //create the counter state machine which holds the counter value
    final CounterStateMachine counterStateMachine = new CounterStateMachine();

    //build the Raft server
    this.server = RaftServer.newBuilder()
        .setGroup(Constants.RAFT_GROUP)
        .setProperties(properties)
        .setServerId(peer.getId())
        .setStateMachine(counterStateMachine)
        .build();
}

public void start() throws IOException {
    server.start();
  }
```

首先看一下RaftServer类是怎么构造的，以及具体的实现类是什么。以下代码来自RaftServer接口

```java
public RaftServer build() throws IOException {
      return newRaftServer(
          serverId,
          group,
          option,
          Objects.requireNonNull(stateMachineRegistry , "Neither 'stateMachine' nor 'setStateMachineRegistry' " +
              "is initialized."),
          Objects.requireNonNull(properties, "The 'properties' field is not initialized."),
          parameters);
}

private static RaftServer newRaftServer(RaftPeerId serverId, RaftGroup group, RaftStorage.StartupOption option,
        StateMachine.Registry stateMachineRegistry, RaftProperties properties, Parameters parameters)
        throws IOException {
      try {
        return (RaftServer) NEW_RAFT_SERVER_METHOD.invoke(null,
            serverId, group, option, stateMachineRegistry, properties, parameters);
      } catch (IllegalAccessException e) {
        throw new IllegalStateException("Failed to build " + serverId, e);
      } catch (InvocationTargetException e) {
        throw IOUtils.asIOException(e.getCause());
      }
}
```

NEW_RAFT_SERVER_METHOD是Method类型，为ServerImplUtils.newRaftServer()方法

```java
  public static RaftServerProxy newRaftServer(
      RaftPeerId id, RaftGroup group, RaftStorage.StartupOption option, StateMachine.Registry stateMachineRegistry,
      RaftProperties properties, Parameters parameters) throws IOException {
    RaftServer.LOG.debug("newRaftServer: {}, {}", id, group);
    if (group != null && !group.getPeers().isEmpty()) {
      Preconditions.assertNotNull(id, "RaftPeerId %s is not in RaftGroup %s", id, group);
      Preconditions.assertNotNull(group.getPeer(id), "RaftPeerId %s is not in RaftGroup %s", id, group);
    }
    final RaftServerProxy proxy = newRaftServer(id, stateMachineRegistry, properties, parameters);
    proxy.initGroups(group, option);
    return proxy;
  }
```

最终构建RaftServer对象是通过 **final RaftServerProxy proxy = newRaftServer(id, stateMachineRegistry, properties, parameters);** 进行构造的，也就是说RaftServer的实现类是在RaftServerProxy中