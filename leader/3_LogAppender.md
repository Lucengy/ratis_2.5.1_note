## 1. 前言

LogAppender用来代表到一个Follower的请求，其主要提供的Api为

```java
AppendEntriesRequestProto newAppendEntriesRequest(long callId, boolean heartbeat) throws RaftLogIOException;

InstallSnapshotRequestProto newInstallSnapshotNotificationRequest(TermIndex firstAvailableLogTermIndex);

Iterable<InstallSnapshotRequestProto> newInstallSnapshotRequests(String requestId, SnapshotInfo snapshot);
```

其入口方法在run()中，run() --> 调用appendLog(boolean)，在appendLog(boolean)方法中，构造AppendEntriesRequest，调用sendRequest(AppendEntriesRequest request, AppendEntriesRequestProto)方法，使用Grpc发送appendEntries的请求

## 2. LogAppender接口

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

