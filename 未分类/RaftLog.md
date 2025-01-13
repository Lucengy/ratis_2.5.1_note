## 1. RaftLogSequentialOps

有关Runner类，Ratis-406的说法是

```
runSequentially does nothing but only an assertion(asserting only at most one thread at a time).
```

接口方法

```java
  //Used by leader
  long append(long term, TransactionContext transaction) throws StateMachineException;
  //Used by leader
  long append(long term, RaftConfiguration configuration);
  //Used by leader
  long appendMetadata(long term, long commitIndex);
  //Used by the leader and the followers.
  CompletableFuture<Long> appendEntry(LogEntryProto entry);
  //Used by the followers.
  List<CompletableFuture<Long>> append(List<LogEntryProto> entries);
  //Used by the leader and the followers.
  CompletableFuture<Long> truncate(long index);
```

## 2. RaftLog

继承自RaftLogSequentialOps和Closeable接口

以下来自Ratis-122

![1736779628165](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1736779628165.png)

![1736779660412](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1736779660412.png)

在Raft.proto文件中，LogEntry的类型目前有三种，如下所示

```protobuf
message LogEntryProto {
  uint64 term = 1;
  uint64 index = 2;

  oneof LogEntryBody {
    StateMachineLogEntryProto stateMachineLogEntry = 3;
    RaftConfigurationProto configurationEntry = 4;
    MetadataProto metadataEntry = 5;
  }
}
```

针对写请求的logEntry，其类型应该是StateMachineLogEntryProto，这里包含RaftLog的内容，同时，做了StateMachine data和RaftLog data的分离，那么StateMachineLogEntryProto的定义为

```protobuf
essage StateMachineLogEntryProto {
  // TODO: This is not super efficient if the SM itself uses PB to serialize its own data for a
  /** RaftLog entry data */
  bytes logData = 1;
  /**
   * StateMachine entry.
   * StateMachine implementation may use this field to separate StateMachine specific data from the RaftLog data.
   */
  StateMachineEntryProto stateMachineEntry = 2;

  enum Type {
    WRITE = 0;
    DATASTREAM = 1;
  }

  Type type = 13;
  // clientId and callId are used to rebuild the retry cache.
  bytes clientId = 14;
  uint64 callId = 15;
}
```

logData是必有的，这里用来表示的是RaftLog的数据，stateMachineData是可选的，其定义为

```protobuf
message StateMachineEntryProto {
   /**
    * StateMachine specific data which is not written to log.
    * Unlike logEntryData, stateMachineData is managed and stored by the StateMachine but not the RaftLog.
    */
  bytes stateMachineData = 1;
   /**
    * When stateMachineData is missing, it is the size of the serialized LogEntryProto along with stateMachineData.
    * When stateMachineData is not missing, it must be set to zero.
    */
  uint32 logEntryProtoSerializedSize = 2;
}
```

看完这里，再来看RaftLog总的内部类EntryWithData

```java
interface EntryWithData {
    int getSerializedSize();
    LogEntryProto getEntry(TimeDuration timeout) throws RaftLogIOException, TimeoutExcepiton;
}
```

接下来是RaftLog的接口

```java
public interface RaftLog extends RaftLogSequentialOps, Closeable {
    
}
```

