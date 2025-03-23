## 1. whenComplete问题

根据CompletionState的JavaDoc

```
Two methods forms (handle nad whenComplete) support unconditional computation whether the triggering stage completed normally or exceptionally. 

Method exceptionaly supports computation only when the triggering stage completes exceptionally, computing a replacement result, similarly to the java catch keyworkd.

In all other cases, if a stage's computation terminates abruptly with an (unchecked) exception or error, then all dependent stages requiring its completion complete exceptionally as well, with a CompletionException holding the exception as its cause.

If a stage is dependent on both of two stages, and both complete exceptionally, then the CompletionException may correspond to either one of these exceptions;
If a statge is dependent on either of two others, and only on of them completes exceptionally, no guarantees are made about whether the dependent stage completes normmaly or exceptionally.
```



## 2. thenCombine相关方法

根据CompletionState的JavaDoc

```
Returns a new CompletionStage that, when this and the other given stage both complete normally, is executed with the two results as arguments to the supplied function
```

api说明

```java
public interface CompletionStage<T> {
    public <U, V> CompletionStage<V> thenCombine(CompletionStage<? extends U> other, BiFunction<? super T, ? super U, ? extends V> fn);
}
```

## 3. thenCompose相关方法

根据CompletionState的JavaDoc

```
Returns a new CompletionStage that is completed with the same value as the CompletionStage returned by the given function.

When this stage contains completes normally, then given function is invoked with this stage's result as the argument, returning another CompletionStage. When that stage completes normally, the CompletionStage returned by this method si completed with the same value.
```

thenCompose和thenApply方法的区别，见

https://stackoverflow.com/questions/43019126/completablefuture-thenapply-vs-thencompose

尽管Joe C描述的更加简洁，但是Kamlesh描述的更加清晰