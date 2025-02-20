## 1. 说明

CloseAsync接口支持异步调用close()方法，是AutoCloseable的子接口，将close()方法使用default进行了修饰。

```java
public interface CloseAsync<REPLY> extends AutoCloseable {
  /** Close asynchronously. */
  CompletableFuture<REPLY> closeAsync();

  /**
   * The same as {@link AutoCloseable#close()}.
   *
   * The default implementation simply calls {@link CloseAsync#closeAsync()}
   * and then waits for the returned future to complete.
   */
  default void close() throws Exception {
    try {
      closeAsync().get();
    } catch (ExecutionException e) {
      final Throwable cause = e.getCause();
      throw cause instanceof Exception? (Exception)cause: e;
    }
  }
}
```

可以看到，close()方法阻塞调用closeAsync().get()方法，等待异步调用执行完成

