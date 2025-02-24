## 0. 前言

为什么会使用到RaftClientRequest类，以及它是怎么转换成protobuf(grpc)中要传输的序列化后的字节流的，答案在ClientProtoUtils.toRaftClientRequestProto(RaftClientRequest)方法中

```java
static RaftClientRequestProto toRaftClientRequestProto(RaftClientRequest request) {
    final RaftClientRequestProto.Builder b = RaftClientRequestProto.newBuilder()
        .setRpcRequest(toRaftRpcRequestProtoBuilder(request));
    if (request.getMessage() != null) {
      b.setMessage(toClientMessageEntryProtoBuilder(request.getMessage()));
    }

    final RaftClientRequest.Type type = request.getType();
    switch (type.getTypeCase()) {
      case WRITE:
        b.setWrite(type.getWrite());
        break;
      case DATASTREAM:
        b.setDataStream(type.getDataStream());
        break;
      case FORWARD:
        b.setForward(type.getForward());
        break;
      case MESSAGESTREAM:
        b.setMessageStream(type.getMessageStream());
        break;
      case READ:
        b.setRead(type.getRead());
        break;
      case STALEREAD:
        b.setStaleRead(type.getStaleRead());
        break;
      case WATCH:
        b.setWatch(type.getWatch());
        break;
      default:
        throw new IllegalArgumentException("Unexpected request type: " + request.getType()
            + " in request " + request);
    }

    return b.build();
  }
```

有关RaftrClientRequestProto的定义如下

```protobuf
message RaftClientRequestProto {
  RaftRpcRequestProto rpcRequest = 1;
  ClientMessageEntryProto message = 2;

  oneof Type {
    WriteRequestTypeProto write = 3;
    ReadRequestTypeProto read = 4;
    StaleReadRequestTypeProto staleRead = 5;
    WatchRequestTypeProto watch = 6;
    MessageStreamRequestTypeProto messageStream = 7;
    DataStreamRequestTypeProto dataStream = 8;
    ForwardRequestTypeProto forward = 9;
  }
}
```

leader在收到RaftClientRequestProto时，转换为RaftClientRequest对象，这部分逻辑在GrpcClientProtocolService的内部类RequestStreamObserver.onNext()方法中

```java
    @Override
    public void onNext(RaftClientRequestProto request) {
      try {
        final RaftClientRequest r = ClientProtoUtils.toRaftClientRequest(request);
        processClientRequest(r);
      } catch (Exception e) {
        responseError(e, () -> "onNext for " + ClientProtoUtils.toString(request) + " in " + name);
      }
    }
```

有关ClientProtoUtils中的toRaftClientRequest(RaftClientRequestProto)方法，如下

```java
static RaftClientRequest toRaftClientRequest(RaftClientRequestProto p) {
    final RaftClientRequest.Type type = toRaftClientRequestType(p);
    final RaftRpcRequestProto request = p.getRpcRequest();

    final RaftClientRequest.Builder b = RaftClientRequest.newBuilder();

    final RaftPeerId perrId = RaftPeerId.valueOf(request.getReplyId());
    if (request.getToLeader()) {
      b.setLeaderId(perrId);
    } else {
      b.setServerId(perrId);
    }
    return b.setClientId(ClientId.valueOf(request.getRequestorId()))
        .setGroupId(ProtoUtils.toRaftGroupId(request.getRaftGroupId()))
        .setCallId(request.getCallId())
        .setMessage(toMessage(p.getMessage()))
        .setType(type)
        .setSlidingWindowEntry(request.getSlidingWindowEntry())
        .setRoutingTable(getRoutingTable(request))
        .setTimeoutMs(request.getTimeoutMs())
        .build();
  }
```



## 1. 祖宗接口RaftRpcMessage

只有四个简单的api

```java
public interface RaftRpcMessage {

  boolean isRequest();

  String getRequestorId();

  String getReplierId();

  RaftGroupId getRaftGroupId();
}

```

## 2. 抽象类RaftClientMessage

一个简单的POJO类，封装了RPC请求的双端peerID，客户端ID使用ClientId对象表示，服务端ID使用RaftPeerId表示，同时持有一个RaftGroupId，用来表示本message所在的RaftGroup

```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.apache.ratis.protocol;

import org.apache.ratis.util.JavaUtils;
import org.apache.ratis.util.Preconditions;

public abstract class RaftClientMessage implements RaftRpcMessage {
  private final ClientId clientId;
  private final RaftPeerId serverId;
  private final RaftGroupId groupId;
  private final long callId;

  RaftClientMessage(ClientId clientId, RaftPeerId serverId, RaftGroupId groupId, long callId) {
    this.clientId = Preconditions.assertNotNull(clientId, "clientId");
    this.serverId = Preconditions.assertNotNull(serverId, "serverId");
    this.groupId = Preconditions.assertNotNull(groupId, "groupId");
    this.callId = callId;
  }

  @Override
  public String getRequestorId() {
    return clientId.toString();
  }

  @Override
  public String getReplierId() {
    return serverId.toString();
  }

  public ClientId getClientId() {
    return clientId;
  }

  public RaftPeerId getServerId() {
    return serverId;
  }

  @Override
  public RaftGroupId getRaftGroupId() {
    return groupId;
  }

  public long getCallId() {
    return callId;
  }

  @Override
  public String toString() {
    return JavaUtils.getClassSimpleName(getClass()) + ":" + clientId + "->" + serverId
        + (groupId != null? "@" + groupId: "") + ", cid=" + getCallId();
  }
}

```

## 3. 实现类RaftClientRequest

先记录RaftClientRequest的定义，它并没有实现SlidingWindow.ClientSideRequest接口

根据RaftClientRequestProto中的定义，RaftClientRequest目前有7种类型

```protobuf
message RaftClientRequestProto {
  RaftRpcRequestProto rpcRequest = 1;
  ClientMessageEntryProto message = 2;

  oneof Type {
    WriteRequestTypeProto write = 3;
    ReadRequestTypeProto read = 4;
    StaleReadRequestTypeProto staleRead = 5;
    WatchRequestTypeProto watch = 6;
    MessageStreamRequestTypeProto messageStream = 7;
    DataStreamRequestTypeProto dataStream = 8;
    ForwardRequestTypeProto forward = 9;
  }
}
```

### 1. 内部类Type

value-based class，持有连个实例变量，分别为

* RaftClientRequestProto.TypeCase typeCase
* Object proto

其中，typeCase对应着protobuf中Type的一种，在用protobuf生成的RaftProtos.java中，TypeCase定义如下

```java
public enum TypeCase {
    WRITE(3),
    READ(4),
    STALEREAD(5),
    WATCH(6),
    MESSAGESTREAM(7),
    DATASTREAM(8),
    FORWARD(9),
    TYPE_NOT_SET(0);
    
    private final int value;
    private TypeCase(int value) {
        this.value = value;
    }
}
```

而proto对象就是对应的WriteRequestTypeProto等对象。

### 2. 内部类Builder

主要看builder()方法的返回值，用来构造的是RaftCilentRequest对象

### 3. RaftClientRequest相关内容

这里关于实例变量boolean toLeader，根据RATIS-1392所描述

```
The leader information can be specified when building a RaftClient. If the leader information is missing, the RaftClient will make a guess and use one of the highest priority servers as the initial leader.

The JIRA proposes to have a leader cache to record the leader information for each group. Then a new RaftClient can use the leader information from the cache so that it don't have to guess who is the leader.
```

在RaftClientRequest中加入toLeader变量，用来标识此request是否发送给leader，如果是，则将leader信息在缓存在RaftClientImpl中，以便后续的Request可以方便的找到leader

* tips 过期读这种类型的request是没必要发送到leader上的吧

RaftClientImpl的代码变动有助于理解，使用了CACHE，将leader信息缓存

![1740095880688](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1740095880688.png)

