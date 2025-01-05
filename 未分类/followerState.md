需要关注的点有三个

## 1. UpdateType

```java
  enum UpdateType {
    APPEND_START(AtomicInteger::incrementAndGet),
    APPEND_COMPLETE(AtomicInteger::decrementAndGet),
    INSTALL_SNAPSHOT_START(AtomicInteger::incrementAndGet),
    INSTALL_SNAPSHOT_COMPLETE(AtomicInteger::decrementAndGet),
    INSTALL_SNAPSHOT_NOTIFICATION(AtomicInteger::get),
    REQUEST_VOTE(AtomicInteger::get);

    private final ToIntFunction<AtomicInteger> updateFunction;

    UpdateType(ToIntFunction<AtomicInteger> updateFunction) {
      this.updateFunction = updateFunction;
    }

    int update(AtomicInteger outstanding) {
      return updateFunction.applyAsInt(outstanding);
    }
  }
```

对于ToIntFunction函数式接口的定义为

```java
@FunctionalInterface
public interface ToIntFunction<T> {

    /**
     * Applies this function to the given argument.
     *
     * @param value the function argument
     * @return the function result
     */
    int applyAsInt(T value);
}
```

类名::实例方法，第一个参数作为方法的调用者，其他参数作为方法的实参

原始写法应该为

```java
public class Impl implements ToIntFunction<AtomicInteger> {
    @Override
    public int applyAsInt(AtomicInteger ai) {
        return ai.get();
        // return ai.incrementAndGet();
    }
}
```

FollowerState中有一个实例变量outstandingOp，用来表示此follower正在执行的任务数量。根据RATIS-443，若其不为0，根据解释，此follower正在

```
to indicate that the follower is writing log/statemachine data. So that it should not time out to start a leader election.
```

那么，其不应该changeToFollower，这在FollowerState.run()方法中有所体现

## 2. lostMajorityHeartbeatsRecently

RATIS-1112，目前不太了解，只是知道其也是作为该follower should changeTo candidate的一个判断叫天

## 3. run() method

FollowerState是一个Daemon线程类，其逻辑均在run()方法中，这里需要专注的只有RaftServerImpl.changeToCandidate() 方法