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

## 3. DataStreamPacketImpl抽象类

根据proto文件，这里的Type类型包含两种，分别是Header信息和Data信息

```protobuf
message DataStreamPacketHeaderProto {
  enum Type {
    STREAM_HEADER = 0;
    STREAM_DATA = 1;
  }

  enum Option {
    SYNC = 0;
    CLOSE = 1;
  }

  bytes clientId = 1;
  Type type = 2;
  uint64 streamId = 3;
  uint64 streamOffset = 4;
  uint64 dataLength = 5;
  repeated Option options = 6;
}
```

DataStreamPacketImpl只是简单的POJO类，封装了clientId，type，streamId和streamOffset信息

```java
public abstract class DataStreamPacketImpl implelmentsw DataStreamPacket {
    private final ClinetId clientId;
    private final Type type;
    private final long streamId;
    private final long  streamOffset;
}
```

## 4. DataStreampPacketHeader

根据上文，DataStreamPacket包含两种类型，分别是Header信息和Data信息。而DataStreampPacketHeader类用来表示Header信息，根据JavaDoc

The header format is streamId, streamOffset, dataLength

```java
public class DataStreamPacketHeader implements DataStreamPacketImpl {
    private static final SizeInBytes SIZE_OF_HEADER_LEN = SizeInBytes.valueOf(4);
  	private static final SizeInBytes SIZE_OF_HEADER_BODY_LEN = SizeInBytes.valueOf(8);

  	private final long dataLength;
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



