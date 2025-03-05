## 1. 前言

nanos是纳秒，millisecond是毫秒，1ms=10<sup>6</sup>ns。构造器中传入的ms。这里还有一点，就是Timestamp是一个value-based类，那么其在构造之后不可变，其内部实例变量nanos也不可变。

elapsedTime 经过的时间

```
This is a value-based class.
```

## 2. 代码

```java
public final class Timestamp implements Comparable<Timestamp> {
    private static final long NANOSECONDS_PER_MILLISECOND = 1000000;
    private final long nanos;
    
    public Timestamp(long nanos) {
        this.nanos = nanos;
    }
    
    //这个也是神奇，虽然是实例方法，但是返回一个全新的对象，并不是调用者本身
    public Timestamp addTimeMs(long milliseconds) {
        return new Timestamp(nanos + milliseconds * NANOSECONDS_PER_MISLLISECOND);
    }
    
    public Timestamp addTime(TimeDuration t) {
        return new Timestamp(nanos + t.to(TimeUnit.NANOSECONDS).getDuration());
    }
    
    public long elapsedTimeMs() {
        final long d = System.nanoTime() - nanos;
        return d / NANOSECONDS_PER_MILLISECOND
    }
    
    @Override
    public int compareTo(Timestamp that) {
        final long d = this.nanos - that.nanos;
        return d > 0? 1: d == 0? 0: -1;
    }
    
    private static final long START_TIME = System.nanoTime(); //在log中使用
    
    public static TimeStamp valueOf(long nanos) {
        return new Timestamp(nanos);
    }
    
    public static TimeStamp currentTimeNanos() {
        return System.nanoTime();
    }
    
    public static Timestamp currentTime() {
        return valueOf(currentTimeNanos());
    }
    
    public static Timestamp lastest(Timestamp a, Timestamp b) {
        return a.compareTo(b) > 0? a: b;
    }
}
```

