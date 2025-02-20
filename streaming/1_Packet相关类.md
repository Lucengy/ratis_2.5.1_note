## 1. 前言

DataStreamPacket是一个接口类，用来描述的是发送的最小单位packet的元数据信息，包含两个子接口，分别是代表请求的DataStreamRequest和代表响应的DataStreamReply接口

## 2. DataStreamPacket接口

```java
public interface DataStreamPacket {
  ClientId getClientId();

  Type getType();

  long getStreamId();

  long getStreamOffset();

  long getDataLength();
}
```

## 3. DataStreamRequest

```java
public interface DataStreamRequest extends DataStreamPacket {
  WriteOption[] getWriteOptions();
}
```

## 4. DataStreamReply

```java
public interface DataStreamReply extends DataStreamPacket {

  boolean isSuccess();

  long getBytesWritten();

  /** @return the commit information when the reply is created. */
  Collection<CommitInfoProto> getCommitInfos();
}
```



