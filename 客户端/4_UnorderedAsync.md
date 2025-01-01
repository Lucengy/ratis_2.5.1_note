## 1. 前言

在研读RrpcClientRpc类时，发现一个令人费解的情况，就是其封装了三个有关RPC的方法，分别为

* sendRequestAsync(RaftClientRequest)
* sendRequestAsyncUnordered(RaftClientRequest)
* sendRequest(RaftClinetRequest)

这三个方法的调用方分别为

* OrderedAsync
* UnorderedAsynsc
* BlockingImpl

为了一探究竟，先从最简单的UnorderedAsync类开始探索。再次强调，这里的unordered是指在发送request时，不需要对其进行排序，也就是需要SlidingWindow的参与，所以相对于ordered，整体要简单很多

## 2. 内部类PendingClientRequest

见PendingClientRequest.md

## 3. UnorderedAsync

只有两个方法

* send()
* sendRequestWithRetry()

首先，来看send()方法中的前部分，CallId.getAndIncrement()是一个static method，用来自增生成clientId。构造PendingUnorderedRequest对象，该对象的构造器主要看其实参Supplier\<RaftClientRequest>，这里是通过client.newRaftClientRequest()方法构造新的RaftClientRequest对象。接下来，将目光转到sendReqeustWithRetry()方法。



```java
  	static CompletableFuture<RaftClientReply> send(RaftClientRequest.Type type, Message message, RaftPeerId server,
      RaftClientImpl client) {
    final long callId = CallId.getAndIncrement();
    final PendingClientRequest pending = new PendingUnorderedRequest(
        () -> client.newRaftClientRequest(server, callId, message, type, null));
    sendRequestWithRetry(pending, client);
    return pending.getReplyFuture()
        .thenApply(reply -> RaftClientImpl.handleRaftException(reply, CompletionException::new));
  }  
static void sendRequestWithRetry(PendingClientRequest pending, RaftClientImpl client) {
    final CompletableFuture<RaftClientReply> f = pending.getReplyFuture();
    if (f.isDone()) {
      return;
    }

    final RaftClientRequest request = pending.newRequest();
    final int attemptCount = pending.getAttemptCount();

    final ClientId clientId = client.getId();
    LOG.debug("{}: attempt #{} send~ {}", clientId, attemptCount, request);
    client.getClientRpc().sendRequestAsyncUnordered(request).whenCompleteAsync((reply, e) -> {
      try {
        LOG.debug("{}: attempt #{} receive~ {}", clientId, attemptCount, reply);
        final RaftException replyException = reply != null? reply.getException(): null;
        reply = client.handleLeaderException(request, reply);
        if (reply != null) {
          client.handleReply(request, reply);
          f.complete(reply);
          return;
        }

        final Throwable cause = replyException != null ? replyException : e;
        pending.incrementExceptionCount(cause);
        final ClientRetryEvent event = new ClientRetryEvent(request, cause, pending);
        RetryPolicy retryPolicy = client.getRetryPolicy();
        final RetryPolicy.Action action = retryPolicy.handleAttemptFailure(event);
        TimeDuration sleepTime = client.getEffectiveSleepTime(cause, action.getSleepTime());
        if (!action.shouldRetry()) {
          f.completeExceptionally(client.noMoreRetries(event));
          return;
        }

        if (e != null) {
          if (LOG.isTraceEnabled()) {
            LOG.trace(clientId + ": attempt #" + attemptCount + " failed~ " + request, e);
          } else {
            LOG.debug("{}: attempt #{} failed {} with {}", clientId, attemptCount, request, e);
          }
          e = JavaUtils.unwrapCompletionException(e);

          if (e instanceof IOException) {
            if (e instanceof NotLeaderException) {
              client.handleNotLeaderException(request, (NotLeaderException) e, null);
            } else if (e instanceof GroupMismatchException) {
              f.completeExceptionally(e);
              return;
            } else {
              client.handleIOException(request, (IOException) e);
            }
          } else {
            if (!client.getClientRpc().handleException(request.getServerId(), e, false)) {
              f.completeExceptionally(e);
              return;
            }
          }
        }

        LOG.debug("schedule retry for attempt #{}, policy={}, request={}", attemptCount, retryPolicy, request);
        client.getScheduler().onTimeout(sleepTime,
            () -> sendRequestWithRetry(pending, client), LOG, () -> clientId + ": Failed~ to retry " + request);
      } catch (Exception ex) {
        LOG.error(clientId + ": Failed " + request, ex);
        f.completeExceptionally(ex);
      }
    });
  }
```

