## 1. 前言

RoleInfo中存在三个状态相关的类，分别是

* LeaderStateImpl
* FollowerState
* LeaderElection

跟Follower相关的为FollowSate类

## 2. UpdateType内部类

美枚举类型，用来表示正在进行的任务，果然，update()方法中对于形参的命名使用的是outgoning

```java
enum UpdateType {
    APPEND_START(AtomicInteger::incrementAndGet),
    APPEND_COMPLETE(AtomicInteger::decrementAndGet),
    INSTALL_SNAPSHOT_START(AtomicInteger::incrementAndGet),
    INSTALL_SNAPSHOT_COMPLETE(AtomicInteger::decrementAndGet),
    INSTALL_SNAPSHOT_NOTIFICATION(AtomicInteger::get),
    REQUEST_VOTE(AtomicInteger::get);

    private final ToIntFunction<AtomicInteger> updateFunction;

//    public static void test() {
//      ToIntFunction<AtomicInteger> t = new ToIntFunction<AtomicInteger>() {
//        @Override
//        public int applyAsInt(AtomicInteger value) {
//          return value.get();
//        }
//      };
//      System.out.println(t.applyAsInt(new AtomicInteger(20)));
//    }

    UpdateType(ToIntFunction<AtomicInteger> updateFunction) {
      this.updateFunction = updateFunction;
    }

    int update(AtomicInteger outstanding) {
      return updateFunction.applyAsInt(outstanding);
    }
  }
```

## 3. FollowState类

### 1. 实例变量

这里需要注意的是，在UpdateType这个枚举类中，并没有提供AtomicInteger对象。使用外部类FollowerState中的实例变量outstandingOp作为实参，传入对应的UpdateType.update()方法中即可

```java
  private final String name;
  private final Object reason;
  private final RaftServerImpl server;

  private final Timestamp creationTime = Timestamp.currentTime();
  private volatile Timestamp lastRpcTime = creationTime;
  private volatile boolean isRunning = true;
  //未完成的操作
  private final AtomicInteger outstandingOp = new AtomicInteger();
```

### 2. 构造器

只是持有一个RaftServerImpl对象

```java
FollowerState(RaftServerImpl server, Object reason) {
    this.name = server.getMemberId() + "-" + JavaUtils.getClassSimpleName(getClass());
    this.setName(this.name);
    this.server = server;
    this.reason = reason;
  }
```

### 3. 相关方法

这里首先要说明的是updateLastRpcTime(UpdateType)方法。这里在嗲用updateLastRpcTime()方法时，意味着一定有新的RPC的到来，那么对应的要更新正在执行的任务的数量，即outstandingOp。这需要根据Rpc的类型进行更新，即这里的形参UpdateType对象

```java
void updateLastRpcTime(UpdateType type) {
    lastRpcTime = Timestamp.currentTime();
    
    final int n = type.update(outstanding);
    
    //LOG相关
}
```

接下来是几个简单的getter()方法

```java
  Timestamp getLastRpcTime() {
    return lastRpcTime;
  }

  int getOutstandingOp() {
    return outstandingOp.get();
  }

  boolean isCurrentLeaderValid() {
    return lastRpcTime.elapsedTime().compareTo(server.properties().minRpcTimeout()) < 0;
  }

  void stopRunning() {
    this.isRunning = false;
  }
```

FollowerState是Daemon的子类，即后台线程，其run()方法是一个死循环，不断监测自己的状态，在合适的时机触发选举机制，通过SererStateImpl.changeToCandidate()方法

```java
  @Override
  public  void run() {
    //Deviation 偏差
    final TimeDuration sleepDeviationThreshold = server.getSleepDeviationThreshold();
    while (shouldRun()) {
      final TimeDuration electionTimeout = server.getRandomElectionTimeout();
      try {
        final TimeDuration extraSleep = electionTimeout.sleep();
        if (extraSleep.compareTo(sleepDeviationThreshold) > 0) {
          LOG.warn("Unexpected long sleep: sleep {} but took extra {} (> threshold = {})",
              electionTimeout, extraSleep, sleepDeviationThreshold);
          continue;
        }

        if (!shouldRun()) {
          break;
        }
        synchronized (server) {
          if (outstandingOp.get() == 0
              && isRunning && server.getInfo().isFollower()
              && lastRpcTime.elapsedTime().compareTo(electionTimeout) >= 0
              && !lostMajorityHeartbeatsRecently()) {
            LOG.info("{}: change to CANDIDATE, lastRpcElapsedTime:{}, electionTimeout:{}",
                this, lastRpcTime.elapsedTime(), electionTimeout);
            server.getLeaderElectionMetrics().onLeaderElectionTimeout(); // Update timeout metric counters.
            // election timeout, should become a candidate
            server.changeToCandidate(false);
            break;
          }
        }
      } catch (InterruptedException e) {
        LOG.info("{} was interrupted", this);
        LOG.trace("TRACE", e);
        Thread.currentThread().interrupt();
        return;
      } catch (Exception e) {
        LOG.warn("{} caught an exception", this, e);
      }
    }
  }
```

