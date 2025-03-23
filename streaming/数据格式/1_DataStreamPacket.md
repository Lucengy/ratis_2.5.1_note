## 1. 前言

是ratis-Streaming的packet的通用数据结构

```java
public interface DataStreamPacket {
    ClientId getClientId();

    Type getType();

    long getStreamId();

    long getStreamOffset();

    long getDataLength();
}
```

这里需要说明的就是Type，其import为

```javascript
import org.apache.ratis.proto.RaftProtos.DataStreamPacketHeaderProto.Type;
```

根据raft.proto中的定义，这里的Type分为Header和Data两部分

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

## 2. DataStreamPacketImpl

是一个抽象实现类

定义了ClientId，Type，streamId，StreamOffset，没有定义dataLength，所有的实例变量均使用final进行修饰

```java
public abstract class DataStreamPacketImpl implements DataStreamPacket {
    private final ClientId clientId;
    private final Type type;
    private final long streamId;
    private final long StreamOffset;
    
    public DataStreamPacketImpl(ClientId clientId, Type type, long streamId, long streamOffset) {
        this.clientId = clientId;
        this.type = type;
        this.streamId = streamId;
        this.streamOffset = streamOffset;
    }
    
    //四个实例变量的getter()方法
}
```

## 3. DataStreamPacketHeader

定义了dataLength

```java
public class DataStreamPacketHeader extends DataStreamPacketImpl {
    private static final SizeInBytes SIZE_OF_HEADER_LEN = SizeInBytes.valueOf(4);
    private static final SizeInBytes SIZE_OF_HEADER_BODY_LEN = SizeInBytes.valueOf(8);
    
    private final long dataLength;
    
    public DataStreamPacketHeader(ClientId clientId, Type type, long streamId, long streamOffset, long dataLength) {
        super(clientId, type, streamId, streamOffset);
        this.dataLength = dataLength;
    }
    
    public long getDataLength() {
        return dataLength;
    }
}
```

## 4. request相关

### 1. DataStreamRequest

##### 1. WriteOption

接口，但是提供了默认实现

```java
public interface WriteOption {
    static boolean containsOption(WriteOption[] options, WriteOption target) {
        for(WriteOption option : options) {
            //这里用的等于号，那应该是enum枚举类咯
            if(option == target) {
                return true;
            }
        }
        return false;
    }
    
    default boolean isOneOf(WriteOption... options) {
        return containsOptions(options, this);
    }
}
```

##### 2. WriteOption默认实现StandardWriteOption 枚举类

```java
public enum StandardWriteOption implements WriteOption {
    SYNC,
    CLOSE
}
```

DataStreamRequest在DataStreamPacket的基础上，额外提供了一个api

```java
public interface DataStreamRequest extends DataStreamPacket() {
    WriteOption[] getWriteOptions();
}
```

### 2. DataStreamRequestHeader

是DataStreamRequest的实现类，是DataStreamPacket的子类

```java
public class DataStreamReqeustHeader extends DataStreamPacketHeader implements DataStreamRequest {
    private final WriteOption[] options;
    
    public DataStreamRequestHeader(ClientId clientId, Type type, long streamId, long streamOffset, long dataLength, WriteOption... options) {
        super(clientId, type, streamId, steramOffset, dataLength);
        this.options = options;
    }
    
    public WriteOption[] getWriteOptions() {
        return options;
    }
}
```

## 5. reply相关

### 1. DataStreamReply

```java
public interface DataStreamReply extends DataStreamPacket {
    boolean isSuccess();
    
    long getBytesWritten();
    
    Collection<CommitInfoProto> getCommitInfos();
}
```

### 2. 实现类 DataStreamReplyHeader

```java
public class DataStreamHeader extends DataStreamPacketHeader implements DataStreamReply {
    private final long bytesWritten;
    private final boolean success;
    private final Collection<CommitInfoProto> commitInfos;
    
    public DataStreamReplyHeader(ClientId clientId, Type type, long streamId, long streamOffset, long dataLength, long bytesWritten, boolean success,  Collection<CommitInfoProto> commitInfos) {
        super(clientId, type, streamId, streamOffset, dataLength);
        this.bytesWritten = bytesWritten;
        this.success = success;
        this.commitInfos = commitInfos;
    }
    
    //三个实例对象的getter方法
}
```

## 6. DataStreamPacketByteBuffer

这里是DataStreamPacketImpl的子类，意味着不包含dataLength这个实例变量。是一个抽象类，但是没有抽象方法，有两个子类，分别为DataStreamRequestByteBuffer和DataStreamReplyByteBuffer

```java
public abstract class DataStreamPacketByteBuffer extends DataStreamPacketImpl {
    public static final EMPTY_BYTE_BUFFER = ByteBuffer.allocateDirect(0).asReadOnlyBuffer();
    
    private final ByteBuffer buffer;
    
    protected DataStreamPacketByteBuffer(ClientId clientId, Type type, long streamId, long streamOffset, ByteBuffer buffer) {
        super(clientId, type, streamId, streamOffset);
        this.buffer = buffer;
    }
    
    public long getDataLength() {
        return buffer.remaining();
    }
    
    public ByteBuffer slice() {
        return buffer.slice();
    }
}
```

### 1. DataStreamRequestByteBuffer

```java
public class DataStreamRequestByteBuffer extends DataStreamPacketByteBuffer implements DataStreamRequest {
    private WriteOption[] options;
    
    public DataStreamReqeustByteBuffer(DataStreamRequestHeader header, ByteBuffer buffer) {
        super(header.getClientId(), header.getType(), header.getStreamId(), header.getStreamOffset(), buffer);
        this.options = options;
    }
    
    public WriteOPtion[] getWriteOptions() {
        return options;
    }
}
```

### 2. DataStreamReplyByteBuffer

```java
public final class DataStreamReplyByteBuffer extends DataStreamPacketByteBuffer implements DataStreamReply {
    public static final class Builder {
        private ClientId clientId;
        private Type type;
        private long streamId;
        private long streamOffset;
        private ByteBuffer buffer;

        private boolean success;
        private long bytesWritten;
        private Collection<CommitInfoProto> commitInfos;

        private Builder() {}

        public Builder setClientId(ClientId clientId) {
            this.clientId = clientId;
            return this;
        }

        public Builder setType(Type type) {
            this.type = type;
            return this;
        }

        public Builder setStreamId(long streamId) {
            this.streamId = streamId;
            return this;
        }

        public Builder setStreamOffset(long streamOffset) {
            this.streamOffset = streamOffset;
            return this;
        }

        public Builder setBuffer(ByteBuffer buffer) {
            this.buffer = buffer;
            return this;
        }

        public Builder setSuccess(boolean success) {
            this.success = success;
            return this;
        }

        public Builder setBytesWritten(long bytesWritten) {
            this.bytesWritten = bytesWritten;
            return this;
        }

        public Builder setCommitInfos(Collection<CommitInfoProto> commitInfos) {
            this.commitInfos = commitInfos;
            return this;
        }

        public Builder setDataStreamReplyHeader(DataStreamReplyHeader header) {
            return setDataStreamPacket(header)
                .setSuccess(header.isSuccess())
                .setBytesWritten(header.getBytesWritten())
                .setCommitInfos(header.getCommitInfos());
        }

        public Builder setDataStreamPacket(DataStreamPacket packet) {
            return setClientId(packet.getClientId())
                .setType(packet.getType())
                .setStreamId(packet.getStreamId())
                .setStreamOffset(packet.getStreamOffset());
        }

        public DataStreamReplyByteBuffer build() {
            return new DataStreamReplyByteBuffer(
                clientId, type, streamId, streamOffset, buffer, success, bytesWritten, commitInfos);
        }
    }

    public static Builder newBuilder() {
        return new Builder();
    }

    private final boolean success;
    private final long bytesWritten;
    private final Collection<CommitInfoProto> commitInfos;

    @SuppressWarnings("parameternumber")
    private DataStreamReplyByteBuffer(ClientId clientId, Type type, long streamId, long streamOffset, ByteBuffer buffer,
                                      boolean success, long bytesWritten, Collection<CommitInfoProto> commitInfos) {
        super(clientId, type, streamId, streamOffset, buffer);

        this.success = success;
        this.bytesWritten = bytesWritten;
        this.commitInfos = commitInfos != null? commitInfos: Collections.emptyList();
    }

    @Override
    public boolean isSuccess() {
        return success;
    }

    @Override
    public long getBytesWritten() {
        return bytesWritten;
    }

    @Override
    public Collection<CommitInfoProto> getCommitInfos() {
        return commitInfos;
    }

    @Override
    public String toString() {
        return super.toString()
            + "," + (success? "SUCCESS": "FAILED")
            + ",bytesWritten=" + bytesWritten;
    }
}
```



## 7. DataStreamRequestByteBuf

ByteBuf是Netty相关的

```java
public class DataStreamRequestByteBuf extends DataStreamPacketImpl implements DataStreamRequest {
    private final ByteBuf buf;
    private final WriteOption[] options;
    
    public DataStreamRequestByteBuf(ClientId clientId, Type type, long streamId, long streamOffset, ByteBuf buf, WriteOption[] options) {
        super(clientId, type, streamId, streamOffset);
        this.buf = buf != null? buf : Unpooled.EMPTY_BUFFER;
        this.options = options;
    }
    
    public DataStreamRequestByteBuf(DataStreamRequestHeader header, ByteBuf buf) {
        this(header.getClientId, ...);
    }
}
```





























