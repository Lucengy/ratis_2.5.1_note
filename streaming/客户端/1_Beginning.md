## 1. 前言

根据https://issues.apache.org/jira/browse/RATIS-1030，一切的源头还是要从RaftClient开始说起，RaftClient添加了一个新的API

```java
DataStreamApi getdataStreamApi();
```

