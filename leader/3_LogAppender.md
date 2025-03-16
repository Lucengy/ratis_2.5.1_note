## 1. 前言

LogAppender根据JavaDoc描述

```
A {@link LogAppender} is for the leader to send appendEntries to a particular follower.
```

用来代表到一个Follower的请求，其主要提供的Api为

```java
AppendEntriesRequestProto newAppendEntriesRequest(long callId, boolean heartbeat) throws RaftLogIOException;

InstallSnapshotRequestProto newInstallSnapshotNotificationRequest(TermIndex firstAvailableLogTermIndex);

Iterable<InstallSnapshotRequestProto> newInstallSnapshotRequests(String requestId, SnapshotInfo snapshot);
```

其入口方法在run()中，run() --> 调用appendLog(boolean)，在appendLog(boolean)方法中，构造AppendEntriesRequest，调用sendRequest(AppendEntriesRequest request, AppendEntriesRequestProto)方法，使用Grpc发送appendEntries的请求

## 2. LogAppender接口

存在一个常量池变量和一个静态方法，用来构造LogAppender默认实现类，即LogAppenderDefault对象

```java
Class<? extends LogAppender> DEFAULT_CLASS = ReflectionUtils.getClass(
      LogAppender.class.getName() + "Default", LogAppender.class);

static LogAppender newLogAppenderDefault(RaftServer.Division server, LeaderState leaderState, FollowerInfo f) {
    final Class<?>[] argClasses = {RaftServer.Division.class, LeaderState.class, FollowerInfo.class};
    return ReflectionUtils.newInstance(DEFAULT_CLASS, argClasses, server, leaderState, f);
}
```



```java
public interface LogAppender {
    RaftServer.Division getServer();

  /** The same as getServer().getRaftServer().getServerRpc(). */
    default getServerRpc() {
        return getServer().getRaftServer().getServerRpc();
    }

    /** The same as getServer().getRaftLog(). */
    default RaftLog getRaftLog() {
        return getServer().getRaftLog();
    }

    /** Start this {@link LogAppender}. */
    void start();

    /** Is this {@link LogAppender} running? */
    boolean isRunning();

    /** Stop this {@link LogAppender}. */
    void stop();

    /** @return the leader state. */
    LeaderState getLeaderState();

    /** @return the follower information for this {@link LogAppender}. */
    FollowerInfo getFollower();

    /** The same as getFollower().getPeer().getId(). */
    default RaftPeerId getFollowerId() {
        return getFollower().getPeer().getId();
    }

    /** @return the call id for the next {@link AppendEntriesRequestProto}. */
    long getCallId();

    /** @return the a {@link Comparator} for comparing call ids. */
    Comparator<Long> getCallIdComparator();

    /**
   * Create a {@link AppendEntriesRequestProto} object using the {@link FollowerInfo} of this {@link LogAppender}.
   * The {@link AppendEntriesRequestProto} object may contain zero or more log entries.
   * When there is zero log entries, the {@link AppendEntriesRequestProto} object is a heartbeat.
   *
   * @param callId The call id of the returned request.
   * @param heartbeat the returned request must be a heartbeat.
   *
   * @return a new {@link AppendEntriesRequestProto} object.
   */
    AppendEntriesRequestProto newAppendEntriesRequest(long callId, boolean heartbeat) throws RaftLogIOException;

    /** @return a new {@link InstallSnapshotRequestProto} object. */
    InstallSnapshotRequestProto newInstallSnapshotNotificationRequest(TermIndex firstAvailableLogTermIndex);

    /** @return an {@link Iterable} of {@link InstallSnapshotRequestProto} for sending the given snapshot. */
    Iterable<InstallSnapshotRequestProto> newInstallSnapshotRequests(String requestId, SnapshotInfo snapshot);

    /**
   * Should this {@link LogAppender} send a snapshot to the follower?
   *
   * @return the snapshot if it should install a snapshot; otherwise, return null.
   */
    default SnapshotInfo shouldInstallSnapshot() {
        // we should install snapshot if the follower needs to catch up and:
        // 1. there is no local log entry but there is snapshot
        // 2. or the follower's next index is smaller than the log start index
        // 3. or the follower is bootstrapping and has not installed any snapshot yet
        final FollowerInfo follower = getFollower();
        final boolean isFollowerBootstrapping = getLeaderState().isFollowerBootstrapping(follower);
        final SnapshotInfo snapshot = getServer().getStateMachine().getLatestSnapshot();

        if (isFollowerBootstrapping && !follower.hasAttemptedToInstallSnapshot()) {
            if (snapshot == null) {
                // Leader cannot send null snapshot to follower. Hence, acknowledge InstallSnapshot attempt (even though it
                // was not attempted) so that follower can come out of staging state after appending log entries.
                follower.setAttemptedToInstallSnapshot();
            } else {
                return snapshot;
            }
        }

        final long followerNextIndex = getFollower().getNextIndex();
        if (followerNextIndex < getRaftLog().getNextIndex()) {
            final long logStartIndex = getRaftLog().getStartIndex();
            if (followerNextIndex < logStartIndex || (logStartIndex == RaftLog.INVALID_LOG_INDEX && snapshot != null)) {
                return snapshot;
            }
        }
        return null;
    }

    /** Define how this {@link LogAppender} should run. */
    void run() throws InterruptedException, IOException;

    /**
   * Get the {@link AwaitForSignal} for events, which can be:
   * (1) new log entries available,
   * (2) log indices changed, or
   * (3) a snapshot installation completed.
   */
    AwaitForSignal getEventAwaitForSignal();

    /** The same as getEventAwaitForSignal().signal(). */
    default void notifyLogAppender() {
        getEventAwaitForSignal().signal();
    }

    /** Should the leader send appendEntries RPC to the follower? */
    default boolean shouldSendAppendEntries() {
        return hasAppendEntries() || getHeartbeatWaitTimeMs() <= 0;
    }

    /** Does it have outstanding appendEntries? */
    default boolean hasAppendEntries() {
        return getFollower().getNextIndex() < getRaftLog().getNextIndex();
    }

    /** send a heartbeat AppendEntries immediately */
    void triggerHeartbeat() throws IOException;

    /** @return the wait time in milliseconds to send the next heartbeat. */
    default long getHeartbeatWaitTimeMs() {
        final int min = getServer().properties().minRpcTimeoutMs();
        // time remaining to send a heartbeat
        final long heartbeatRemainingTimeMs = min/2 - getFollower().getLastRpcResponseTime().elapsedTimeMs();
        // avoid sending heartbeat too frequently
        final long noHeartbeatTimeMs = min/4 - getFollower().getLastHeartbeatSendTime().elapsedTimeMs();
        return Math.max(heartbeatRemainingTimeMs, noHeartbeatTimeMs);
    }

    /** Handle the event that the follower has replied a term. */
    default boolean onFollowerTerm(long followerTerm) {
        synchronized (getServer()) {
            return isRunning() && getLeaderState().onFollowerTerm(getFollower(), followerTerm);
        }
    }
}
```

## 3. LogAppenderBase抽象类

是LogAppender的一个抽象实现

```java
private final String name;
private final RaftServer.Division server;
private final LeaderState leaderState;
private final FollowerInfo follower;

private final DataQueue<EntryWithData> buffer;
private final int snapshotChunkMaxSize;

private final LogAppenderDaemon daemon;
private final AwaitForSignal eventAwaitForSignal;

private final AtomicBoolean heartbeatTrigger = new AtomicBoolean();
```

有关实例变量，这里需要了解的是LogAppenderDaemon deamon对象

在构造器中将用此对象来构造LogAppenderDaemon对象

```java
this.daemon = new LogAppenderDaemon(this);
```

这里的方法比较简单，主要是三个构造proto对象的方法，以及一个getPrevious(long nextIndex)方法有一点拗口

```java
private TermIndex getPrevious(long nextIndex) {
    if (nextIndex == RaftLog.LEAST_VALID_LOG_INDEX) {
        return null;
    }

    final long previousIndex = nextIndex - 1;
    final TermIndex previous = getRaftLog().getTermIndex(previousIndex);
    if (previous != null) {
        return previous;
    }

    final SnapshotInfo snapshot = server.getStateMachine().getLatestSnapshot();
    if (snapshot != null) {
        final TermIndex snapshotTermIndex = snapshot.getTermIndex();
        if (snapshotTermIndex.getIndex() == previousIndex) {
            return snapshotTermIndex;
        }
    }

    return null;
}
```

这里的getPrevious()方法，首先从rafgLog里面去取

* 如果raftLog能找到，直接返回
* 如果raftLog未能找到，那么去snapshot里面去找

从snapshot里面找的话，存在三种情况

* 没有snapshot，那么直接返回null
* 存在snapshot，且snapshot最后的TermIndex就是自己要找的，那么返回
* 存在snapshot，但是snapshot最后的TermIndex并不是自己要找的，这种情况是出现问题了，返回null

snapshot的TermIndex为其包含的最后一条logEntry的termIndex，在getPrevious()方法中，为什么要对snapshot的TermIndex跟previousIndex进行比较呢？这是因为，考虑逻辑，找上一条logEntry，要么它一定存在于RaftLog中，要么其一定是snapshot的最后一条。为什么是这样呢，因为currentIndex一定是存在于raftLog中的

接下来就是三个构造proto对象的方法

```java

```



### 1. LogAppenderDaemon

持有一个LogAppender对象，构造器对其进行赋值操作

```java
class LogAppenderDaemon {
  public static final Logger LOG = LoggerFactory.getLogger(LogAppenderDaemon.class);

  private final String name;
  private final LifeCycle lifeCycle;
  private final Daemon daemon;

  private final LogAppender logAppender;

  LogAppenderDaemon(LogAppender logAppender) {
    this.logAppender = logAppender;
    this.name = logAppender + "-" + JavaUtils.getClassSimpleName(getClass());
    this.lifeCycle = new LifeCycle(name);
    this.daemon = new Daemon(this::run, name);
  }
}
```

持有一个Daemon对象，赋值时，将此对象赋值给Daemon，那么daemon.start()启动时会调用this.run()方法，而this.run()调用的是logAppender.start()方法

```java
private void run() {
    try {
        if (lifeCycle.transition(TRY_TO_RUN) == RUNNING) {
            logAppender.run();
        }
        lifeCycle.compareAndTransition(RUNNING, CLOSING);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        LOG.info(this + " was interrupted: " + e);
    } catch (InterruptedIOException e) {
        LOG.info(this + " I/O was interrupted: " + e);
    } catch (Throwable e) {
        LOG.error(this + " failed", e);
        lifeCycle.transitionIfValid(EXCEPTION);
    } finally {
        if (lifeCycle.transitionAndGet(TRANSITION_FINALLY) == EXCEPTION) {
            logAppender.getLeaderState().restart(logAppender);
        }
    }
}
```

### 2. DataQueue\<E>类

根据JavaDoc描述，队列对元素数量和数据大小（字节）都有限制

```
A queue for data elements such that the queue imposes limits on both number of elements and the data size in bytes.
```

除去numBytes，其它实例变量都是用final进行修饰。而numBytes做了默认初始化

```java
public class DataQueue<E> implements Iterable<E> {
    private final Object name;
    private final long byteLimit;
    private final ToLongFunction<E> getNumBytes;
    private final Queue<E> q;
    private long numBytes = 0;
}
```

构造器就是简单的赋值

```java
public DataQueue(Object name, SizeInBytes byteLimit, int elementLimit,
      ToLongFunction<E> getNumBytes) {
    this.name = name != null? name: this;
    this.byteLimit = byteLimit.getSize();
    this.elementLimit = elementLimit;
    this.getNumBytes = getNumBytes;
    this.q = new LinkedList<>();
}
```

存在几个简单的getter()方法

```java
public int getElementLimit() {
    return elementLimit;
}

public long getByteLimit() {
    return byteLimit;
}

public long getNumBytes() {
    return numBytes;
}

public int getNumElements() {
    return q.size();
}
```

判断队列为空以及清空队列的两个方法

```java
public final boolean isEmpty() {
    return getNumElements() == 0;
}

/** The same as {@link java.util.Collection#clear()}. */
public void clear() {
    q.clear();
    numBytes = 0;
}
```

入队列方法。首先，入队列的元素不能为null值；其次，对其sizeLimit和countLimit进行判断，符合条件做入队列操作

```java
public boolean offer(E element) {
    Objects.requireNonNull(element, "element == null");
    if (elementLimit > 0 && q.size() >= elementLimit) {
        return false;
    }
    final long elementNumBytes = getNumBytes.applyAsLong(element);
    Preconditions.assertTrue(elementNumBytes >= 0,
                             () -> name + ": elementNumBytes = " + elementNumBytes + " < 0");
    if (byteLimit > 0) {
        Preconditions.assertTrue(elementNumBytes <= byteLimit,
                                 () -> name + ": elementNumBytes = " + elementNumBytes + " > byteLimit = " + byteLimit);
        if (numBytes > byteLimit - elementNumBytes) {
            return false;
        }
    }
    q.offer(element);
    numBytes += elementNumBytes;
    return true;
}
```

出队列方法，当队列为空时，会返回空值

```java
public E poll() {
    final E polled = q.poll();
    if (polled != null) {
        numBytes -= getNumBytes.applyAsLong(polled);
    }
    return polled;
}
```

还有一个通用的出队列方法，这里的返回值并不是Element，而是一个List\<Element>，根据JavaDoc，是给定timeout，在timeout之内返回一个resultList

```java
public <RESULT, THROWABLE extends Throwable> List<RESULT> pollList(long timeoutMs, CheckedFuntionWithTimeout<E, RESULT, THROWABLE> getResult, TriConsumer<E, TimeDuration, TimeoutException> timeoutHandler) throws THROWABLE {
    if(timeoutMs <= 0 || q.isEmpty()) {
        return Collections.emptyList();
    }
    final Timestamp startTime = Timestamp.currentTime();
    final TimeDuration limit = TimeDuration.valueOf(timeoutMs, TimeUnit.MILLISECONDS);
    for(final List<RESULT> results = new ArrayList<>();;) {
        final E peeked = q.peek();
        if (peeked == null) { // q is empty
            return results;
        }

        final TimeDuration remaining = limit.subtract(startTime.elapsedTime());
        try {
            results.add(getResult.apply(peeked, remaining));
        } catch (TimeoutException e) {
            Optional.ofNullable(timeoutHandler).ifPresent(h -> h.accept(peeked, remaining, e));
            return results;
        }

        final E polled = poll();
        Preconditions.assertTrue(polled == peeked);
    }
}
```



删除操作

```java
public boolean remove(E e) {
    final boolean removed = q.remove(e);
    if (removed) {
        numBytes -= getNumBytes.applyAsLong(e);
    }
    return removed;
}
```

迭代器

```java
@Override
public Iterator<E> iterator() {
    final Iterator<E> i = q.iterator();
    // Do not support the remove() method.
    return new Iterator<E>() {
        @Override
        public boolean hasNext() {
            return i.hasNext();
        }

        @Override
        public E next() {
            return i.next();
        }
    };
}
```

## 4. 实现类GrpcLogAppender

### 1. AppendEntriesRequest

简单的一个POJO类

```java
static class AppendEntriesRequest {
    private final Timer timer;
    private volatile Timer.Context timerContext;

    private final long callId;
    private final TermIndex previousLog;
    private final int entriesCount;

    private final TermIndex lastEntry;

    AppendEntriesRequest(AppendEntriesRequestProto proto, RaftPeerId followerId, GrpcServerMetrics grpcServerMetrics) {
        this.callId = proto.getServerRequest().getCallId();
        this.previousLog = proto.hasPreviousLog()? TermIndex.valueOf(proto.getPreviousLog()): null;
        this.entriesCount = proto.getEntriesCount();
        this.lastEntry = entriesCount > 0? TermIndex.valueOf(proto.getEntries(entriesCount - 1)): null;

        this.timer = grpcServerMetrics.getGrpcLogAppenderLatencyTimer(followerId.toString(), isHeartbeat());
        grpcServerMetrics.onRequestCreate(isHeartbeat());
    }

    long getCallId() {
        return callId;
    }

    TermIndex getPreviousLog() {
        return previousLog;
    }

    void startRequestTimer() {
        timerContext = timer.time();
    }

    void stopRequestTimer() {
        timerContext.stop();
    }

    boolean isHeartbeat() {
        return entriesCount == 0;
    }

    @Override
    public String toString() {
        return JavaUtils.getClassSimpleName(getClass())
            + ":cid=" + callId
            + ",entriesCount=" + entriesCount
            + ",lastEntry=" + lastEntry;
    }
}
```

### 2. RequestsMap

封装了两个map，分别是logMap和heartbeatMap，其中key为callID，value为AppendEntiresRequest

```java
static class RequestMap {
    private final Map<Long, AppendEntriesRequest> logRequests = new ConcurrentHashMap<>();
    private final Map<Long, AppendEntriesRequest> heartbeats = new ConcurrentHashMap<>();

    int logRequestsSize() {
        return logRequests.size();
    }

    void clear() {
        logRequests.clear();
        heartbeats.clear();
    }

    void put(AppendEntriesRequest request) {
        if (request.isHeartbeat()) {
            heartbeats.put(request.getCallId(), request);
        } else {
            logRequests.put(request.getCallId(), request);
        }
    }

    AppendEntriesRequest remove(AppendEntriesReplyProto reply) {
        return remove(reply.getServerReply().getCallId(), reply.getIsHearbeat());
    }

    AppendEntriesRequest remove(long cid, boolean isHeartbeat) {
        return isHeartbeat ? heartbeats.remove(cid): logRequests.remove(cid);
    }

    public AppendEntriesRequest handleTimeout(long callId, boolean heartbeat) {
        return heartbeat ? heartbeats.remove(callId) : logRequests.get(callId);
    }
}
```

### 3. AppendLogResponseHandler

存在两个Handler对象，分别为AppendLogResponseHandler和InstallSnpashotResponseHandler对象，都继承自StreamObserver对象，用来监听RPC的响应信息

#### AppendLogResponseHandler类

首先看onNext方法的JavaDoc

```java
/**
     * After receiving a appendEntries reply, do the following:
     * 1. If the reply is success, update the follower's match index and submit
     *    an event to leaderState
     * 2. If the reply is NOT_LEADER, step down
     * 3. If the reply is INCONSISTENCY, increase/ decrease the follower's next
     *    index based on the response
     */
```

这里需要打一个问号，第二种情况，当返回值为NOT_LEADER时，这个是告诉本leader，在下已经退位的意思嘛？

```java
private class AppendLogResponseHandler implements StreamObserver<AppendEntriesReplyProto> {
    public void onNext(AppendEntriesReplyProto reply) {
        //首先从缓存的发送列表移除这个request。这里remove()的实参是一个reply对象
        //这是因为remove方法只需要知道callId即可，request和reply中都携带着这部分信息
        AppendEntriesRequest reqeust = pendingRequests.remove(reply);
        if(request != null) {
            request.stopRequestTimer();
        }
        
        //LOG
        
        try {
            onNextImpl(reply);
        } catch(Exception i) {
            //LOG
        }
    }
    
    private void onNextImpl(AppendEntriesReplyProto reply) {
        // update the last rpc time
        getFollower().updateLastRpcResponseTime();

        if (!firstResponseReceived) {
            firstResponseReceived = true;
        }

        switch (reply.getResult()) {
            case SUCCESS:
                grpcServerMetrics.onRequestSuccess(getFollowerId().toString(), reply.getIsHearbeat());
                //1. 通知leader
                getLeaderState().onFollowerCommitIndex(getFollower(), reply.getFollowerCommit());
                //2. 更新leader中的followerInfo信息
                if (getFollower().updateMatchIndex(reply.getMatchIndex())) {
                    getLeaderState().onFollowerSuccessAppendEntries(getFollower());
                }
                break;
            case NOT_LEADER:
                grpcServerMetrics.onRequestNotLeader(getFollowerId().toString());
                if (onFollowerTerm(reply.getTerm())) {
                    return;
                }
                break;
            case INCONSISTENCY:
                grpcServerMetrics.onRequestInconsistency(getFollowerId().toString());
                updateNextIndex(reply.getNextIndex());
                break;
            default:
                throw new IllegalStateException("Unexpected reply result: " + reply.getResult());
        }
        getLeaderState().onAppendEntriesReply(getFollower(), reply);
        notifyLogAppender();
    }
}
```

针对1，在reply返回成功时，需要通知leader

```java
@Override
public void onError(Throwable t) {
    if (!isRunning()) {
        LOG.info("{} is already stopped", GrpcLogAppender.this);
        return;
    }
    GrpcUtil.warn(LOG, () -> this + ": Failed appendEntries", t);
    grpcServerMetrics.onRequestRetry(); // Update try counter
    AppendEntriesRequest request = pendingRequests.remove(GrpcUtil.getCallId(t), GrpcUtil.isHeartbeat(t));
    resetClient(request, true);
}

@Override
public void onCompleted() {
    LOG.info("{}: follower responses appendEntries COMPLETED", this);
    resetClient(null, false);
}
```

这里的onError()方法和onCompleted()方法都调用了外部类，即GrpcLogAppender的resetClient方法

```java
private void resetClient(AppendEntriesRequest request, boolean onError) {
    try (AutoCloseableLock writeLock = lock.writeLock(caller, LOG::trace)) {
        getClient().resetConnectBackoff();
        appendLogRequestObserver = null;
        firstResponseReceived = false;
        // clear the pending requests queue and reset the next index of follower
        pendingRequests.clear();
        final long nextIndex = 1 + Optional.ofNullable(request)
            .map(AppendEntriesRequest::getPreviousLog)
            .map(TermIndex::getIndex)
            .orElseGet(getFollower()::getMatchIndex);
        if (onError && getFollower().getMatchIndex() == 0 && request == null) {
            LOG.warn("{}: Leader has not got in touch with Follower {} yet, " +
                     "just keep nextIndex unchanged and retry.", this, getFollower());
            return;
        }
        getFollower().decreaseNextIndex(nextIndex);
    } catch (IOException ie) {
        LOG.warn(this + ": Failed to getClient for " + getFollowerId(), ie);
    }
}
```

#### InstallSnpashotResponseHandler类