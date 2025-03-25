## 1. 前言

最早是由RATIS-122提出来的，在FileStore的例子中，使用了stateMachine去写data，而不是raftLog

```
The file data are stored by the state machine (but not in the raft log).
```

