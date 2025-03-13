## 1. 前言

## 2. 静态内部类Permit

空对象

```java
static class Permit {}
```

## 3. 静态内部类RequestLimits

是ResourceSemaphore.Group的子类，Group类对list的长度没有限制，只是对使用进行了限制，即

* 第一个元素必须是个数上的限制
* 其他元素必须是字节数上的限制

而RequestLimits是对list的长度加以限制，只有两个元素。这样便为

* 第一个元素是元素个数上的限制
* 第二个元素是字节大小上的限制

```java
static class RequestLimits extends ResourceSemaphore.Group {
    RequestLimits(int elementLimit, int megabyteLimit) {
        super(elementLimit, megabyteLimit);
    }
    
    int getElementCount() {
        return get(0).used();
    }
    
    int getMegaByteSize() {
        return get(1).used();
    }
    
    /**
    这个方法比较拗口。我申请占用一个元素，同时申请占用messageSizeMb的字节数
    **/
    ResourceSemaphore.ResourceAcquireStatus tryAcquire(int messageSizeMb) {
        return tryAcquire(1, messageSizeMb);
    }
    
    //这两个release方法一个没有归还元素个数，一个归还了一个
    void releaseExtraMb(int extraMb) {
        release(0, extraMb);
    }

    void release(int diffMb) {
        release(1, diffMb);
    }
}
```

