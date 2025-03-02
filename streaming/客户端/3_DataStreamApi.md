## 1. 前言

用来获取DataStreamOutput对象

```java
public interface DataStreamApi extends Closeable {
  /** Create a stream to write data. */
  default DataStreamOutput stream() {
    return stream(null);
  }

  /** Create a stream by providing a customized header message. */
  DataStreamOutput stream(ByteBuffer headerMessage);

  /** Create a stream by providing a customized header message and route table. */
  DataStreamOutput stream(ByteBuffer headerMessage, RoutingTable routingTable);
}
```

