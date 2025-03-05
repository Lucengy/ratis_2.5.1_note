## 1. 前言

## 2. RaftServerProtocol vs RaftServerAsynchronousProtocol

### 1. RaftServerProtocol

对应着RaftServerProtocolService中四个rpc的定义

```protobuf
service RaftServerProtocolService {
  rpc requestVote(ratis.common.RequestVoteRequestProto)
      returns(ratis.common.RequestVoteReplyProto) {}

  rpc startLeaderElection(ratis.common.StartLeaderElectionRequestProto)
      returns(ratis.common.StartLeaderElectionReplyProto) {}

  rpc appendEntries(stream ratis.common.AppendEntriesRequestProto)
      returns(stream ratis.common.AppendEntriesReplyProto) {}

  rpc installSnapshot(stream ratis.common.InstallSnapshotRequestProto)
      returns(stream ratis.common.InstallSnapshotReplyProto) {}
}
```

其接口定义为

```java
public interface RaftServerProtocol {
    enum Op {REQUEST_VOTE, APPEND_ENTRIES, INSTALL_SNAPSHOT}

    RequestVoteReplyProto requestVote(RequestVoteRequestProto request) throws IOException;

    AppendEntriesReplyProto appendEntries(AppendEntriesRequestProto request) throws IOException;

    InstallSnapshotReplyProto installSnapshot(InstallSnapshotRequestProto request) throws IOException;

    StartLeaderElectionReplyProto startLeaderElection(StartLeaderElectionRequestProto request) throws IOException;
}
```

### 2. RaftServerAsynchronousProtocol

只是对appendEntries这个rpc的返回值使用CompletableFuture进行了包装

```java
public interface RaftServerAsynchronousProtocol {
  CompletableFuture<AppendEntriesReplyProto> appendEntriesAsync(AppendEntriesRequestProto request)
      throws IOException;
}
```

## 3. RaftClientProtocol vs RaftClientAsynchronousProtocol

只是submitClientRequest()方法的同步和异步版本

### 1. RaftClientProtocol

```java
public interface RaftClientProtocol {
  RaftClientReply submitClientRequest(RaftClientRequest request) throws IOException;
}
```

### 2. RaftClientAsynchronousProtocol

```java
public interface RaftClientAsynchronousProtocol {
  CompletableFuture<RaftClientReply> submitClientRequestAsync(
      RaftClientRequest request) throws IOException;
}
```

## 4. AdminProtocol vs AdminAsynchronousProtocol

### 1. AdminProtoocl

对应这AdminProtooclService中7个rpc的定义

```protobuf
service AdminProtocolService {
  // A client-to-server RPC to set new raft configuration
  rpc setConfiguration(ratis.common.SetConfigurationRequestProto)
      returns(ratis.common.RaftClientReplyProto) {}

  rpc transferLeadership(ratis.common.TransferLeadershipRequestProto)
      returns(ratis.common.RaftClientReplyProto) {}

  // A client-to-server RPC to add a new group
  rpc groupManagement(ratis.common.GroupManagementRequestProto)
      returns(ratis.common.RaftClientReplyProto) {}

  rpc snapshotManagement(ratis.common.SnapshotManagementRequestProto)
      returns(ratis.common.RaftClientReplyProto) {}

  rpc leaderElectionManagement(ratis.common.LeaderElectionManagementRequestProto)
      returns(ratis.common.RaftClientReplyProto) {}

  rpc groupList(ratis.common.GroupListRequestProto)
      returns(ratis.common.GroupListReplyProto) {}

  rpc groupInfo(ratis.common.GroupInfoRequestProto)
      returns(ratis.common.GroupInfoReplyProto) {}
}
```

接口定义如下

```java
public interface AdminProtocol {
  GroupListReply getGroupList(GroupListRequest request) throws IOException;

  GroupInfoReply getGroupInfo(GroupInfoRequest request) throws IOException;

  RaftClientReply groupManagement(GroupManagementRequest request) throws IOException;

  RaftClientReply snapshotManagement(SnapshotManagementRequest request) throws IOException;

  RaftClientReply leaderElectionManagement(LeaderElectionManagementRequest request) throws IOException;

  RaftClientReply setConfiguration(SetConfigurationRequest request) throws IOException;

  RaftClientReply transferLeadership(TransferLeadershipRequest request) throws IOException;
}
```

### 2. AdminAsynchronousProtoocl

是上述7个rpc的异步版本，返回值使用CompletableFuture进行修饰

```java
public interface AdminAsynchronousProtocol {
  CompletableFuture<GroupListReply> getGroupListAsync(GroupListRequest request);

  CompletableFuture<GroupInfoReply> getGroupInfoAsync(GroupInfoRequest request);

  CompletableFuture<RaftClientReply> groupManagementAsync(GroupManagementRequest request);

  CompletableFuture<RaftClientReply> snapshotManagementAsync(SnapshotManagementRequest request);

  CompletableFuture<RaftClientReply> leaderElectionManagementAsync(LeaderElectionManagementRequest request);

  CompletableFuture<RaftClientReply> setConfigurationAsync(
      SetConfigurationRequest request) throws IOException;

  CompletableFuture<RaftClientReply> transferLeadershipAsync(
      TransferLeadershipRequest request) throws IOException;
}
```

