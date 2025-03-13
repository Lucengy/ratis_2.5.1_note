## 1. 前言

ResourceSemaphore继承自 Semaphore类，基本上没做什么事情，只是对其及进行了简单的封装

## 2. ResourceSemaphore

存在三个实例变量，均使用final进行修饰

* final int limit
* final AtomicBoolean reducePermits = new AtomicBoolean()
* final AtomicBoolean isClosed = new AtomicBoolean()

reducePermits和isClosed都是默认赋值的，这里get()方法均返回false

构造器也是调用了父类构造器，同时加入了assert断言

```java
public ResourceSemaphore(int limit) {
    super(limit, true);
    Preconditions.assertTure(limit > 0);
    this.limit = limit;
}
```

实例方法，首先要看的也是两个assert()方法，这里只是做例行检查

```java
private int assertAvailable {
    final int available = availablePermits();
    Preconditions.assertTrue(available >= 0, () -> "available = " + available + " < 0");
    return avaliable;
}

private void assertRelease(int toRelease) {
    Preconditions.assertTrue(toRelease >= 0, () -> "toRelease = " + toRelease + " < 0");
    final int available = assertAvailable();
    final int permits = Math.addExact(available, toRelease);
    Preconditions.assertTrue(permits <= limit, () -> "permits = " + permits + " > limit = " + limit);
}
```

releaser()方法，在调用super.release()方法前，也做了断言判断

```java
@Override
public void release() {
    release(1);
}

@Override
public void release(int permits) {
    assertRelease(permits);
    super.release(permits);
    assertAvailable();
}
```

这里提供了used()方法，也很简单

```java
public int used() {
    return limit - availablePermits();
}
```

使用reducePermits去调控isClosed实例变量

```java
public void close() {
    if (reducePermits.compareAndSet(false, true)) {
        reducePermits(limit);
        isClosed.set(true);
    }
}

public boolean isClosed() {
    return isClosed.get();
}
```

## 3. 内部枚举类ResourceAcquireStatus

```java
public enum ResourceAcquireStatus {
    SUCCESS,
    FAILED_IN_ELEMENT_LIMIT,
    FAILED_IN_BYTE_SIZE_LIMIT
}
```

## 4. 静态内部类Group

持有一个ResourceSemaphore的list，使用final修饰该list

```java
public static class Group {
    private final List<ResourceSemaphore> resources;
    
    public Group(int... limits) {
        Preconditions.assertTrue(limits.length >= 1, () -> "limits is empty");
        final List<ResourceSemaphore> list = new ArrayList<>(limits.length);
        for(int limit : limits) {
            list.add(new ResourceSemaphore(limit));
        }
        
        this.resources = Collections.unmodifiableList(list);
    }
    
    public int resourceSize() {
        return resources.size();
    }
    
    protected ResourceSemaphore get(int i) {
      return resources.get(i);
    }
    
    public ResourceAcquireStatus tryAcquire(int... permits) {
        Preconditions.assertTrue(permits.length == resources.size(),
          () -> "items.length = " + permits.length + " != resources.size() = " + resources.size());
        
        int i = 0;
        for(; i < premits.length; i ++) {
            if(!resources.get(i).tryAcquire(permits[i])) {
                break;
            }
        }
        
        if (i == permits.length) {
        	return ResourceAcquireStatus.SUCCESS; // successfully acquired all resources
      	}
		
        //这里默认李彪第一个为数量上的限制，其他为字节数上的限制
        ResourceAcquireStatus acquireStatus;
        if (i == 0) {
            acquireStatus =  ResourceAcquireStatus.FAILED_IN_ELEMENT_LIMIT;
        } else {
            acquireStatus =  ResourceAcquireStatus.FAILED_IN_BYTE_SIZE_LIMIT;
        }

        // failed at i, releasing all previous resources
        //当失败时，释放已经获取到的信号量
        for(i--; i >= 0; i--) {
            resources.get(i).release(permits[i]);
        }

        return acquireStatus;
    }
    
    protected void release(int... premits) {
        for(int i = resources.size() - 1; i >= 0; i --) {
            resources.get(i).release(permits[i]);
        }
    }
    
    public void close() {
        for(int i = resources.size() - 1; i >= 0; i--) {
            resources.get(i).close();
        }
    }
    
    public boolean isClosed() {
        return resources.get(resources.size() - 1).isClosed();
    }
}
```



