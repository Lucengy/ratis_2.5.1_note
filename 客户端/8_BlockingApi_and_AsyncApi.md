## 1. 前言

之前一直关注Rpc方面，并没有过多的关注客户端这边涉及到的一些类。回到最初的CounterClient

```java
if (blocking) {
      // use BlockingApi
      final ExecutorService executor = Executors.newFixedThreadPool(10);
      for (int i = 0; i < increment; i++) {
        final Future<RaftClientReply> f = executor.submit(
            () -> client.io().send(CounterCommand.INCREMENT.getMessage())); //a
        futures.add(f);
      }
      executor.shutdown();
    } else {
      // use AsyncApi
      for (int i = 0; i < increment; i++) {
        final Future<RaftClientReply> f = client.async().send(CounterCommand.INCREMENT.getMessage()); //b
        futures.add(f);
      }
    }
```

这里的client有两种调用方式，分别是client.io()--阻塞方法以及client.async()--异步方法。这里涉及到的两个类，分别为

* BlockingApi
* AsyncApi

这里关注这两个接口及其实现类

## 2. BlockingApi && AsyncApi

可以看到，BlockingApi和AsyncApi中方法的功能一摸一样，只是api的返回值有所区别。其对应四个功能，分别为

* send() 发送写请求
* sendReadOnly() 发送读请求
* sendStaleRead() 过期读请求
* watch() 观察请求

1. BlockingApi

```java
public interface BlockingApi {
    RaftClientReply send(Message message) throws IOException;
    
    RaftClientReply sendReadOnly(Message message) throws IOException;
    
    RaftClientReply sendStaleRead(Message message, long minIndex, RaftPeerId server) throws IOException;
    
    RaftClientReply watch(long index, ReplicationLevel replication) throws IOException;
}
```

2. AsyncApi

```java
public interface AsyncApi {
    CompletableFuture<RaftClientReply> send(Message message);
    
    CompletableFuture<RaftClientReply> sendReadOnly(Message message);
    
    CompletableFuture<RaftClientReply> sendStaleRead(Message message, long minIndex, RaftPeerId server);
    
    CompletableFuture<RaftClientReply> watch(long index, ReplicationLevel replication);
}
```

## 3. BlockingApi实现BlockingImpl

所有的send()相关方法，最终调用的是send(RaftClientRequest.Type, Message, RaftPeerId)重载方法

```java
private RaftClientReply send(RaftClientRequest.Type type, Message message, RaftPeerId server)
      throws IOException {
    if (!type.is(TypeCase.WATCH)) {
        Objects.requireNonNull(message, "message == null");
    }

    final long callId = CallId.getAndIncrement();
    return sendRequestWithRetry(() -> client.newRaftClientRequest(server, callId, message, type, null));
}
```

该方法调用sendRequestWithRetry()方法

```java
RaftClientReply sendRequestWithRetry(Supplier<RaftClientRequest> supplier) throws IOException {
    RaftClientImpl.PendingClientRequest pending = new RaftClientImpl.PendingClientRequest() {
        @Override
        public RaftClientRequest newRequestImpl() {
            return supplier.get();
        }
    };
    while (true) {
        final RaftClientRequest request = pending.newRequest();
        IOException ioe = null;
        try {
            final RaftClientReply reply = sendRequest(request);

            if (reply != null) {
                return client.handleReply(request, reply);
            }
        } catch (GroupMismatchException | StateMachineException | TransferLeadershipException |
                 LeaderSteppingDownException | AlreadyClosedException | AlreadyExistsException |
                 SetConfigurationException e) {
            throw e;
        } catch (IOException e) {
            ioe = e;
        }

        pending.incrementExceptionCount(ioe);
        ClientRetryEvent event = new ClientRetryEvent(request, ioe, pending);
        final RetryPolicy retryPolicy = client.getRetryPolicy();
        final RetryPolicy.Action action = retryPolicy.handleAttemptFailure(event);
        TimeDuration sleepTime = client.getEffectiveSleepTime(ioe, action.getSleepTime());

        if (!action.shouldRetry()) {
            throw (IOException)client.noMoreRetries(event);
        }

        try {
            sleepTime.sleep();
        } catch (InterruptedException e) {
            throw new InterruptedIOException("retry policy=" + retryPolicy);
        }
    }
}
```

最红调用sendRequest(RaftClientRequest)方法。该方法通过RaftClientRpc.sendRequest(RaftClientRequest)方法将请求发送给服务端

```java
private RaftClientReply sendRequest(RaftClientRequest request) throws IOException {
    LOG.debug("{}: send {}", client.getId(), request);
    RaftClientReply reply;
    try {
        reply = client.getClientRpc().sendRequest(request);
    } catch (GroupMismatchException gme) {
        throw gme;
    } catch (IOException ioe) {
        client.handleIOException(request, ioe);
        throw ioe;
    }
    LOG.debug("{}: receive {}", client.getId(), reply);
    reply = client.handleLeaderException(request, reply);
    reply = RaftClientImpl.handleRaftException(reply, Function.identity());
    return reply;
}
```

## 4. AsyncApi实现AsyncImpl

因为Ratis-Streaming的缘故，这里AsyncApi又出现了一个新的子接口，即AsyncRpcApi，定义了新的sendForward(RaftClientRequest)方法

```java
public interface AsyncRpcApi extends AsyncApi {
    /**
   * Send the given forward-request asynchronously to the raft service.
   *
   * @param request The request to be forwarded.
   * @return a future of the reply.
   */
    CompletableFuture<RaftClientReply> sendForward(RaftClientRequest request);
}
```

在AsyncImpl中，watchRequest使用的是UnorderedAsync进行发送的，其他使用OrderedAsync进行发送的