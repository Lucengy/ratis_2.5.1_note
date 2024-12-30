首先从GrpcClientProtocolClient 类开始讲起

## 1. 前言

在Grpc-java中，客户端使用(blocking)stub发起RPC请求，GrpcClientProtocolClient 中封装了RaftClientProtocolService和AdminProtocolService，前者是给client做数据stream使用的，后者是admin接口，主要是用来查看集群状态，以及对集群状态改变，如transferLeader、changeConf等操作，如Grpc.proto中所示

```java
service RaftClientProtocolService {
  // A client-to-server stream RPC to ordered async requests
  rpc ordered(stream ratis.common.RaftClientRequestProto)
      returns (stream ratis.common.RaftClientReplyProto) {}

  // A client-to-server stream RPC for unordered async requests
  rpc unordered(stream ratis.common.RaftClientRequestProto)
      returns (stream ratis.common.RaftClientReplyProto) {}
}

service AdminProtocolService {
  // A client-to-server RPC to set new raft configuration
  rpc setConfiguration(ratis.common.SetConfigurationRequestProto)
      returns(ratis.common.RaftClientReplyProto) {}

  rpc transferLeadership(ratis.common.TransferLeadershipRequestProto)
      returns(ratis.common.RaftClientReplyProto) {}

  // A client-to-server RPC to add a new group
  rpc groupManagement(ratis.common.GroupManagementRequestProto)
      returns(ratis.common.RaftClientReplyProto) {}

  rpc snapshotManagement(ratis.common.SnapshotManagementRequestProto)
      returns(ratis.common.RaftClientReplyProto) {}

  rpc leaderElectionManagement(ratis.common.LeaderElectionManagementRequestProto)
      returns(ratis.common.RaftClientReplyProto) {}

  rpc groupList(ratis.common.GroupListRequestProto)
      returns(ratis.common.GroupListReplyProto) {}

  rpc groupInfo(ratis.common.GroupInfoRequestProto)
      returns(ratis.common.GroupInfoReplyProto) {}
}
```

## 2. field

如前言中所说，GrpcClientProtocolClient封装了RaftClientProtocolService和AdminProtocolService，那么对应的，就要有二者的stub对象和StreamObserver对象，同时，可以根据配置决定admin接口和cilent接口是否复用同一连接。对Grpc中的Channel概念不是很熟，但是在Netty中，Channel代表着一个Socket连接，盲目类比到Grpc中，成为了是否复用同一个Socket连接

* RaftClientProtocolServiceStub asyncStub
* AdminProtocolServiceBlocking adminBlockingStub
* AtomicReference\<AsyncStreamObservers> orderedStreamObservers
* AtomicReference\<AsyncStreamObservers> unorderedStreamObservers
* ManagedChannel clientChannel
* ManagedChannel adminChannel

首先看构造器中clientChannel和adminChannel的初始化

```java
	final boolean separateAdminChannel = !Objects.equals(clientAddress, adminAddress);    
	clientChannel = buildChannel(clientAddress, clientTlsConfig,
        flowControlWindow, maxMessageSize);
    adminChannel = separateAdminChannel
        ? buildChannel(adminAddress, adminTlsConfig,
            flowControlWindow, maxMessageSize)
        : clientChannel;
```

可以看到，是根据配置文件来决定复用与否的

接下来，是stub的初始化，client使用的是异步的stub，admin使用的是同步的stub，这也对应着前面的Grpc.proto文件，service RaftClientProtocolService为Bidirectional 类型，而service AdminProtocolService为unary类型

```java
    asyncStub = RaftClientProtocolServiceGrpc.newStub(clientChannel);
    adminBlockingStub = AdminProtocolServiceGrpc.newBlockingStub(adminChannel);
```

