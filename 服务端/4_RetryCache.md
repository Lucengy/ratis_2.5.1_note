RetryCache用来缓存客户端的答复，能够快速应答客户端重试的请求

## 1. RetryCache接口

两个内部类，Entry和Statistics。Entry封装了请求ID和对应的Reply信息；Statistics用来记录缓存命中信息

```java
interface Entry {
    ClientInvocationId getKey();
    CompletableFuture<RaftClientReply> getReplyFuture();
}
```

```java
interface Statistics {
    long size(); //返回缓存中RaftClientReply的数量
    long hitCount(); //返回缓存命中次数
    double hitRate(); //返回缓存命中百分比
    long missCount();
    doulbe missRate();
}
```

整个RetryCache的接口方法比较简单

```java
interface Retrycache extends Closeable {
    Entry getIfPresent(ClientInvocationId key);
    Statistics getStatistics();
}
```

## 2. RetryCacheImpl实现类

相应的，先看Entry和Statistics的实现类

1. CacheEntry类

   理所应当的，持有两个实例变量，分别为ClientInvocationId和CompletableFuture\<RaftClientReply>，同时，持有一个boolean值变量，用来表示该RaftClientRequest是否已经失败了。注意这里的replyFuture的类型是被CompletableFuture引用的，同时，replyFuture是默认初始化的，所以在构造器中只有对key的初始化，关于这个future的信息，是后续通过调用updateResult(RaftClinetReply)等方法进行更新的

```java
  static class CacheEntry implements Entry {
      private final ClientInvocationId key;
      private final CompletableFuture<RaftClientReply> replyFuture = new CompletableFuture<>();
      private volatile boolean failed = false;
      
      CacheEntry(ClientInvocationId key) {
      	this.key = key;
      }
      
      boolean isDone() {
      	return isFailed() || replyFuture.isDone();
      }
      
      boolean isCompletedNormally() {
      	return !failed && replyFuture.isDone() && !replyFuture.isCompletedExceptionally() && 				!replyFuture.isCancelled();
      }
      
      void updateResult(RaftClientReply reply) {
      	assert !replyFuture.isDone() && !replyFuture.isCancelled();
      	replyFuture.complete(reply);
      }
      
      boolean isFailed() {
      	return failed || replyFuture.isCompletedExceptionally();
      }
      
      void failWithReply(RaftClientReply reply) {
      	failed = true;
      	replyFuture.complete(reply);
      }
      
      void failWithException(Throwable t) {
      	failed = true;
      	replyFuture.completeExceptionally(t);
   	  }
      
      //override的两个getter()
  }
```

2. CacheQueryResult

   暂时不知道这个类的用途

   ```java
     static class CacheQueryResult {
       private final CacheEntry entry;
       private final boolean isRetry;
   
       CacheQueryResult(CacheEntry entry, boolean isRetry) {
         this.entry = entry;
         this.isRetry = isRetry;
       }
   
       public CacheEntry getEntry() {
         return entry;
       }
   
       public boolean isRetry() {
         return isRetry;
       }
     }
   ```

RetryCacheImpl使用Cache类缓存了一个ClientInvocationId-->CacheEntry的map，构造器也是对cache进行初始化

```java
  private final Cache<ClientInvocationId, CacheEntry> cache;

  RetryCacheImpl(TimeDuration cacheExpiryTime, TimeDuration statisticsExpiryTime) {
    this.cache = CacheBuilder.newBuilder()
        .recordStats()
        .expireAfterWrite(cacheExpiryTime.getDuration(), cacheExpiryTime.getUnit())
        .build();
    this.statisticsExpiryTime = statisticsExpiryTime;
  }
```

需要琢磨一点的是这个getOrCreateEntry(ClientInvocationId)方法。这里当cache miss掉ClinetIncoationId时，put了一个新的CacheEntry对象，这个新的CacheEntry对象是调用CacheEntry的构造器构建的，通过上文CacheEntry的构造器可知，此时新的CacheEntry中的replyFuture对象只是一个空的CompletableFuture

```java
  CacheEntry getOrCreateEntry(ClientInvocationId key) {
    final CacheEntry entry;
    try {
      entry = cache.get(key, () -> new CacheEntry(key));
    } catch (ExecutionException e) {
      throw new IllegalStateException(e);
    }
    return entry;
  }
```

核心的的方法为queryCache(ClinetInvocationId)方法

```java
CacheQueryResult queryCache(ClientInvocationId key) {
    final CacheEntry newEntry = new CacheEntry(key);
    final CacheEntry cacheEntry;
    try {
      cacheEntry = cache.get(key, () -> newEntry);
    } catch (ExecutionException e) {
      throw new IllegalStateException(e);
    }

    if (cacheEntry == newEntry) {
      // this is the entry we just newly created
      return new CacheQueryResult(cacheEntry, false);
    } else if (!cacheEntry.isDone() || !cacheEntry.isFailed()){
      // the previous attempt is either pending or successful
      return new CacheQueryResult(cacheEntry, true);
    }

    // the previous attempt failed, replace it with a new one.
    synchronized (this) {
      // need to recheck, since there may be other retry attempts being
      // processed at the same time. The recheck+replacement should be protected
      // by lock.
      final CacheEntry currentEntry = cache.getIfPresent(key);
      if (currentEntry == cacheEntry || currentEntry == null) {
        // if the failed entry has not got replaced by another retry, or the
        // failed entry got invalidated, we add a new cache entry
        return new CacheQueryResult(refreshEntry(newEntry), false);
      } else {
        return new CacheQueryResult(currentEntry, true);
      }
    }
  }
```

