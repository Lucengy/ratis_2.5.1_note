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

## 3. method

有关AdminProtocolService的方法先一笔带过，后面再深入研究，这里只是提一下blockingCall(CheckdedSuppiler\<RaftClientReplyProto, StatusRutimeException>)方法，其实我觉得没有什么意义，只是看起来高大上，因为GrpcClientProtocolClient 本身是blocking 的RPC

```java
  private static RaftClientReplyProto blockingCall(
      CheckedSupplier<RaftClientReplyProto, StatusRuntimeException> supplier
      ) throws IOException {
    try {
      return supplier.get();
    } catch (StatusRuntimeException e) {
      throw GrpcUtil.unwrapException(e);
    }
  }
```

所有的AdminProtocolService相关的RPC均调用了blockingCall方法，以snapshotManagement(SnapshotManagementRequestProto)为例

```java
  RaftClientReplyProto snapshotManagement(
      SnapshotManagementRequestProto request) throws IOException {
    return blockingCall(() -> adminBlockingStub
        .withDeadlineAfter(requestTimeoutDuration.getDuration(), requestTimeoutDuration.getUnit())
        .snapshotManagement(request));
  }
```

接下来，重点关注RaftClientProtocolService的相关方法，主要是两个ordered()以及unordered()两个RPC方法，这里需要注意的是，ordered()方法是给async IO使用的，unordered()方法是给blocking IO使用的，其在GrpcClientProtocolClient 中的定义相对简单

```java
  StreamObserver<RaftClientRequestProto> ordered(StreamObserver<RaftClientReplyProto> responseHandler) {
    return asyncStub.ordered(responseHandler);
  }

  StreamObserver<RaftClientRequestProto> unorderedWithTimeout(StreamObserver<RaftClientReplyProto> responseHandler) {
    return asyncStub.withDeadlineAfter(requestTimeoutDuration.getDuration(), requestTimeoutDuration.getUnit())
        .unordered(responseHandler);
  }
```

RPC中的StreamObserver有两个，一个是你提供给RPC方法的，即这里的responseHandler，用来作为输入流的回调函数；一个是RPC方法的返回值，作为输出流的回调函数。这里对于输入流输出流的处理逻辑，均由调用方提供，需要注意的是，unordered()是给blocking  IO使用的，但是这里使用的依旧是async，即异步的方法，所以blocking IO的概念是由上层调用方提供的。这里需要先看一下内部类部分，再向下看接下来的两个方法

## 4. 内部类

1. replyMap

   封装了一个map，key为RequestID，value为CompletableFuture\<RaftClientReply>

   ```java
       private final AtomicReference<Map<Long, CompletableFuture<RaftClientReply>>> map
           = new AtomicReference<>(new ConcurrentHashMap<>());
   ```

   主要是两个方法，一个是putNew()，一个是getAndSetNull()。需要琢磨的地方是，getAndSetNull()方法，针对的不是requestID，而是整个map，这里的map是一个atomicReference类型，同时从getAndSetNull()的返回值是一个Map类型这两个点去体会该方法返回值是一个map，同时将AtomicReference置为null

   ```java
       synchronized CompletableFuture<RaftClientReply> putNew(long callId) {
         return Optional.ofNullable(map.get())
             .map(m -> CollectionUtils.putNew(callId, new CompletableFuture<>(), m, this::toString))
             .orElse(null);
       }
   
       synchronized Map<Long, CompletableFuture<RaftClientReply>> getAndSetNull() {
         return map.getAndSet(null);
       }
   ```

2. RequestStreamer

   封装了一个StreamObserver，需要琢磨的地方是，StreamObserver中的泛型为RaftClientRequestProto，我们处在client侧，泛型为request类型，那么此StreamObserver为输出流的处理逻辑

   ```java
     	private final AtomicReference<StreamObserver<RaftClientRequestProto>> streamObserver;
       
   	RequestStreamer(StreamObserver<RaftClientRequestProto> streamObserver) {
         this.streamObserver = new AtomicReference<>(streamObserver);
       }
   ```

   使用synchronized同步了onNext()方法和onComplete()方法，主要的处理逻辑是在onComplete()方法中，将staremObserver置为null；同时，在onNext()方法中，加入了streamObserver为null的判断逻辑

   ```java
       synchronized boolean onNext(RaftClientRequestProto request) {
         final StreamObserver<RaftClientRequestProto> s = streamObserver.get();
         if (s != null) {
           s.onNext(request);
           return true;
         }
         return false;
       }
   
       synchronized void onCompleted() {
         final StreamObserver<RaftClientRequestProto> s = streamObserver.getAndSet(null);
         if (s != null) {
           s.onCompleted();
         }
       }
   ```

3. AsyncStreamObservers

   AsyncStreamObservers对象是统一的接口，里面封装了一个RequestStreamer对象，即输出流的处理逻辑；封装了一个StreamObserver\<RaftClientReplyProto>对象，即输出流的处理逻辑；同时封装了一个replyMap，即所有request的对应的CompletableFuture\<RaftClientReply>，用来检测request请求是否已经完成。

   关于RequestStreamer对象和replyMap对象不再赘述，关注点在持有一个StreamObserver\<RaftClientReplyProto>对象，根据泛型类型，加之我们处在client侧，可以判断此StreamObserver为输入流的处理逻辑

   ```java
     private final RequestStreamer requestStreamer;
     private final ReplyMap replies = new ReplyMap();
    
     private final StreamObserver<RaftClientReplyProto> replyStreamObserver
           = new StreamObserver<RaftClientReplyProto>() {
         @Override
         public void onNext(RaftClientReplyProto proto) {
           final long callId = proto.getRpcReply().getCallId();
           try {
             final RaftClientReply reply = ClientProtoUtils.toRaftClientReply(proto);
             LOG.trace("{}: receive {}", getName(), reply);
             final NotLeaderException nle = reply.getNotLeaderException();
             if (nle != null) {
               completeReplyExceptionally(nle, NotLeaderException.class.getName());
               return;
             }
             final LeaderNotReadyException lnre = reply.getLeaderNotReadyException();
             if (lnre != null) {
               completeReplyExceptionally(lnre, LeaderNotReadyException.class.getName());
               return;
             }
             handleReplyFuture(callId, f -> f.complete(reply));
           } catch (Exception e) {
             handleReplyFuture(callId, f -> f.completeExceptionally(e));
           }
         }
   
         @Override
         public void onError(Throwable t) {
           final IOException ioe = GrpcUtil.unwrapIOException(t);
           completeReplyExceptionally(ioe, "onError");
         }
   
         @Override
         public void onCompleted() {
           completeReplyExceptionally(null, "completed");
         }
       };
   ```

   这里提供了onNext() onError() onCompleted()方法，根据Raft.proto中关于RaftClientReplyProto中的描述，onNext()方法对返回的出错状态进行了逻辑上的处理，在没有出错的情况下，使用handleReplyFuture(callId, f -> f.complete(reply))对返回结果进行处理，可以看到除去completeReplyExceptionally()，需要关注的就剩下handleReplyFuture()方法了，而其中的回调函数f -> f.complete(reply)也相对简单，就是简单的将callId对应的reply的CompletableFuture标注为完成状态，并赋值为对应的reply。

   ```protobuf
   message RaftClientReplyProto {
     RaftRpcReplyProto rpcReply = 1;
     ClientMessageEntryProto message = 2;
   
     oneof ExceptionDetails {
       NotLeaderExceptionProto notLeaderException = 3;
       NotReplicatedExceptionProto notReplicatedException = 4;
       StateMachineExceptionProto stateMachineException = 5;
       LeaderNotReadyExceptionProto leaderNotReadyException = 6;
       AlreadyClosedExceptionProto alreadyClosedException = 7;
       ThrowableProto dataStreamException = 8;
       ThrowableProto leaderSteppingDownException = 9;
       ThrowableProto transferLeadershipException = 10;
       ThrowableProto readException = 11;
       ThrowableProto readIndexException = 12;
     }
   
     uint64 logIndex = 14; // When the request is a write request and the reply is success, the log index of the transaction
     repeated CommitInfoProto commitInfos = 15;
   }
   ```

   首先关注其构造函数

   



