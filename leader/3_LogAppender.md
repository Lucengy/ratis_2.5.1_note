## 1. 前言

LogAppender用来代表到一个Follower的请求，其主要提供的Api为

```java
AppendEntriesRequestProto newAppendEntriesRequest(long callId, boolean heartbeat) throws RaftLogIOException;

InstallSnapshotRequestProto newInstallSnapshotNotificationRequest(TermIndex firstAvailableLogTermIndex);

Iterable<InstallSnapshotRequestProto> newInstallSnapshotRequests(String requestId, SnapshotInfo snapshot);
```

其入口方法在run()中，run() --> 调用appendLog(boolean)，在appendLog(boolean)方法中，构造AppendEntriesRequest，调用sendRequest(AppendEntriesRequest request, AppendEntriesRequestProto)方法，使用Grpc发送appendEntries的请求

