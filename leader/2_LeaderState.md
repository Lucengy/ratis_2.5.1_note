## 0. 前言

方法的入口在RaftServerImpl.changeToLeader() --> RoleInfo.startLeaderState(RaftServerImpl) --> LeaderStateImpl.start()

根据RATIS-353，leader在掌权时，应该发送一个placeHolder的logEntry，之前发送的应该是no-op，即心跳。现在发送的是raftConf。这里注意placeHolder的logIndex使用的是raftLog.getNextIndex()，也对哈，意味着每个leader掌权时都会有一条conf的logEntry，然后在本地raftLog中添加了一条logEntry，调用EventProcessor.start()Daemon线程监测和更新leaderStateImpl的状态信息。senders保存着到各个Follower的Rpc类，调用其start()方法

```java
  LogEntryProto start() {
    // In the beginning of the new term, replicate a conf entry in order
    // to finally commit entries in the previous term.
    // Also this message can help identify the last committed index and the conf.
    final LogEntryProto placeHolder = LogProtoUtils.toLogEntryProto(
        server.getRaftConf(), server.getState().getCurrentTerm(), raftLog.getNextIndex());
    CodeInjectionForTesting.execute(APPEND_PLACEHOLDER,
        server.getId().toString(), null);
    raftLog.append(Collections.singletonList(placeHolder));
    processor.start();
    senders.forEach(LogAppender::start);
    return placeHolder;
  }
```



## 1. 接口

需要学习的是其接口定义，以**on**开头的方法都是事件驱动型的方法，当收到对应事件时，会被调用

```java
public interface LeaderState {
  /** The reasons that this leader steps down and becomes a follower. */
  enum StepDownReason {
    HIGHER_TERM, HIGHER_PRIORITY, LOST_MAJORITY_HEARTBEATS, STATE_MACHINE_EXCEPTION, JVM_PAUSE, FORCE;

    private final String longName = JavaUtils.getClassSimpleName(getClass()) + ":" + name();

    @Override
    public String toString() {
      return longName;
    }
  }

  /** Restart the given {@link LogAppender}. */
  void restart(LogAppender appender);

  /** @return a new {@link AppendEntriesRequestProto} object. */
  AppendEntriesRequestProto newAppendEntriesRequestProto(FollowerInfo follower,
      List<LogEntryProto> entries, TermIndex previous, long callId);

  /** Check if the follower is healthy. */
  void checkHealth(FollowerInfo follower);

  /** Handle the event that the follower has replied a term. */
  boolean onFollowerTerm(FollowerInfo follower, long followerTerm);

  /** Handle the event that the follower has replied a commit index. */
  void onFollowerCommitIndex(FollowerInfo follower, long commitIndex);

  /** Handle the event that the follower has replied a success append entries. */
  void onFollowerSuccessAppendEntries(FollowerInfo follower);

  /** Check if a follower is bootstrapping. */
  boolean isFollowerBootstrapping(FollowerInfo follower);

  /** Received an {@link AppendEntriesReplyProto} */
  void onAppendEntriesReply(FollowerInfo follower, AppendEntriesReplyProto reply);

}
```

## 2. LeaderStateImpl实现

整体比较复杂，其内部类较多。根据JavaDoc描述，这个类包含了三种不同的线程

1. RPC senders: 每个线程负责向一个follower发送appendLog
2. EventProcessor: a single thread，根据rpc结果更新leader的状态信息
3. PendingRequestHandler: 当logEntry成功commit后，使用该线程将结果发送给client