RaftServerImpl持有一个RoleInfo对象，RoleInfo持有一个LeaderStateImpl对象，LeaderStateImpl持有一个SenderList对象，Sendlist为LeaderStateImpl的内部类

```java
static class SenderList {
    private final List<LogAppender> senders;

    SenderList() {
        this.senders = new CopyOnWriteArrayList<>();
    }

    Stream<LogAppender> stream() {
        return senders.stream();
    }

    List<LogAppender> getSenders() {
        return senders;
    }

    void forEach(Consumer<LogAppender> action) {
        senders.forEach(action);
    }

    void addAll(Collection<LogAppender> newSenders) {
        if (newSenders.isEmpty()) {
            return;
        }

        Preconditions.assertUnique(
            CollectionUtils.as(senders, LogAppender::getFollowerId),
            CollectionUtils.as(newSenders, LogAppender::getFollowerId));

        final boolean changed = senders.addAll( );
        Preconditions.assertTrue(changed);
    }

    boolean removeAll(Collection<LogAppender> c) {
        return senders.removeAll(c);
    }
}
```

实际上就是管理着一个LogAppender的list