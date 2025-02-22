## 1. 前言

抽象类，用来标识来自客户端的请求

## 2. 代码注释

```java
  public abstract static class PendingClientRequest {
    private final long creationTimeInMs = System.currentTimeMillis();
    private final CompletableFuture<RaftClientReply> replyFuture = new CompletableFuture<>();
    private final AtomicInteger attemptCount = new AtomicInteger();
    private final Map<Class<?>, Integer> exceptionCount = new ConcurrentHashMap<>();

    public abstract RaftClientRequest newRequestImpl();

    final RaftClientRequest newRequest() {
      attemptCount.incrementAndGet();
      return newRequestImpl();
    }

    CompletableFuture<RaftClientReply> getReplyFuture() {
      return replyFuture;
    }

    public int getAttemptCount() {
      return attemptCount.get();
    }

    int incrementExceptionCount(Throwable t) {
      return t != null ? exceptionCount.compute(t.getClass(), (k, v) -> v != null ? v + 1 : 1) : 0;
    }

    public int getExceptionCount(Throwable t) {
      return t != null ? Optional.ofNullable(exceptionCount.get(t.getClass())).orElse(0) : 0;
    }

    public boolean isRequestTimeout(TimeDuration timeout) {
      if (timeout == null) {
        return false;
      }
      return System.currentTimeMillis() - creationTimeInMs > timeout.toLong(TimeUnit.MILLISECONDS);
    }
  }
```

## 3. 实现类PendingOrderedRequest

OrderedAsyn中的静态内部类



