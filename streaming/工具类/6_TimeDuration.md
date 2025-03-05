## 1. 前言

```
* This is a value-based class.
```

## 2. 代码

### 1. 内部类Abbreviation

这里的name()方法打印的是NANOSECONDS MICROSECONDS等值。这里有一点需要另外说明的是，Abbreviation的构造器中，list在创建的时候多创建了两个，以SECONDS为例，这里symbols中不仅仅包含s, sec，还要包含SECONDS和SECOND；但是以DAYS为例，symbols中包含了day, d，还要包含days。所以symbols至多需要多添加两个

```java
public enum Abbreviation {
    NANOSECONDS("ns", "nanos"),
    MICROSECONDS("μs", "us", "micros"),
    MILLISECONDS("ms", "msec", "millis"),
    SECONDS("s", "sec"),
    MINUTES("min", "m"),
    HOURS("hr", "h"),
    DAYS("day", "d");
    
    private final TimeUnit unit = TimeUnit.valueOf(name());
    
    private final List<String> symbols;
    
    Abbreviation(String... symbols) {
        final List<String> input = Arrays.asList(symbols);
        final List<String> all = new ArrayList<>(input.size() + 2);
        input.forEach(s -> all.add(s.toLowerCase()));
        
        final String s = unit.name().toLowerCase();
        addIfAbsent(all, s);
        addIfAbsent(all, s.substring(0, s.length() - 1));
        this.symbols = Collections.unmodifiableList(all);
    }
    
    static void addIfAbsent(List<String> list, String toAdd) {
      for(String s : list) {
        if (toAdd.equals(s)) {
          return;
        }
      }
      list.add(toAdd);
    }
    
    public TimeUnit unit() {
        return unit;
    }
    
    String getDefault() {
        return symbols.get(0);
    }
    
    public List<String> getSymbols() {
        return symbols;
    }
    
    public static Abbreviation valueOf(TimeUnit unit) {
        return valueOf(unit.name());
    }
}
```

### 2. 

```java
public final class TimeDuration implements Comparable<TimeDuration> {
    public static TimeDuration valueOf(long duration, TimeUnit unit) {
        return new TimeDuration(duration, unit);
    }
    
    private final long duration;
    private final TimeUnit unit;
    
    private TimeDuration(long duration, TimeUnit unit) {
        this.duration = duration;
        this.unit = unit;
    }
    
    /**这里比较拗口，直接上代码就理解了
    * TimeUnit.HOURS.convert(3600, TimeUnit.MINUTES) = 60
    **/
    public long toLong(TimeUnit targetUnit) {
        return targetUnit.convert(duration, unit);
    }
    
    public int toIntExact(TimeUnit targetUnit) {
        return Math.toIntExact(toLong(targetUnit));
    }
    
    public TimeDuration to(TimeUnit targetUnit) {
        if(this.unit == targetUnit) {
            return this;
        }
        
        final TimeDuration t = valueOf(toLong(targetUnit), targetUnit);
        return t;
    }
    
    public TimeDuration add(TimeDuration that) {
        Objects.requireNonNull(that, "that == null");
        final TimeUnit minUnit = ColelctionUtils.min(this.unit, that.unit);
        return valueOf(this.toLong(minUnit) + that.toLong(minUnit), minUnit);
    }
    
    public TimeDuration add(long thatDuration, TimeUnit thatUnit) {
        return add(TimeDuration.valueOf(thatDuration, thatUnit));
    }
    
    public TimeDuration subtract(TimeDuration that) {
        Objects.requireNonNull(that, "that == null");
        final TimeUnit minUnit = CollectionUtils.min(this.unit, that.unit);
        return valueOf(this.toLong(minUnit) - that.toLong(minUnit), minUnit);
    }
}
```

