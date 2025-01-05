接口定义，是Raft leader用来处理client的入口方法

## 1. 接口定义

目前知道这个是异步的就好了，果然有异步，就有同步，同步接口定义为RaftClientProtocol

```java
public interface RaftClientAsynchronousProtocol {
  CompletableFuture<RaftClientReply> submitClientRequestAsync(
      RaftClientRequest request) throws IOException;
}
```

```java
public interface RaftClientProtocol {
  RaftClientReply submitClientRequest(RaftClientRequest request) throws IOException;
}
```

## 华丽的分割线

这接口名定的，乱七八糟啊

```java
public interface RaftServerProtocol {
  enum Op {REQUEST_VOTE, APPEND_ENTRIES, INSTALL_SNAPSHOT}

  RequestVoteReplyProto requestVote(RequestVoteRequestProto request) throws IOException;

  AppendEntriesReplyProto appendEntries(AppendEntriesRequestProto request) throws IOException;

  InstallSnapshotReplyProto installSnapshot(InstallSnapshotRequestProto request) throws IOException;

  StartLeaderElectionReplyProto startLeaderElection(StartLeaderElectionRequestProto request) throws IOException;
}
```

```java
public interface RaftServerAsynchronousProtocol {

  CompletableFuture<AppendEntriesReplyProto> appendEntriesAsync(AppendEntriesRequestProto request)
      throws IOException;

  CompletableFuture<ReadIndexReplyProto> readIndexAsync(ReadIndexRequestProto request)
      throws IOException;
}
```

