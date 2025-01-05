![1736084949298](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1736084949298.png)

![1736084980854](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1736084980854.png)

## 1. RaftStorageDirectory接口

接口，整体代码比较简单，就先全贴过来，这里需要终于注意一点，以getCurrentDir()方法为例，该方法返回的应该是current这一层，那么getRoot()方法返回的是上一层的目录，即UUID那一层目录，而不是n0那一层

```java
public interface RaftStorageDirectory {
  Logger LOG = LoggerFactory.getLogger(RaftStorageDirectory.class);

  String CURRENT_DIR_NAME = "current";
  String STATE_MACHINE_DIR_NAME = "sm"; // directory containing state machine snapshots
  String TMP_DIR_NAME = "tmp";

  /** @return the root directory of this storage */
  File getRoot();

  /** @return the current directory. */
  default File getCurrentDir() {
    return new File(getRoot(), CURRENT_DIR_NAME);
  }

  /** @return the state machine directory. */
  default File getStateMachineDir() {
    return new File(getRoot(), STATE_MACHINE_DIR_NAME);
  }

  /** @return the temporary directory. */
  default File getTmpDir() {
    return new File(getRoot(), TMP_DIR_NAME);
  }

  /** Is this storage healthy? */
  boolean isHealthy();
}
```

## 2. RaftStorageDirectoryImpl

实现类，首先，定义了三个文件，分别为in_use.lock，raft-meta，raft-meta.conf

* String IN_USE_LOCK_NAME = "in_use.lock"
* String META_FILE_NAME = "raft-meta"
* String CONF_EXTENSION = ".conf"
* File root
* FileLock lock
* long freeSpaceMin

然后，定义了root这个实例变量，一个文件锁对象，以及freeSpaceMin实例变量

接下来，主要关注一下构造器，在构造器中，并没有对FileLock进行初始化

```java
  RaftStorageDirectoryImpl(File dir, long freeSpaceMin) {
    this.root = dir;
    this.lock = null;
    this.freeSpaceMin = freeSpaceMin;
  }
```

紧接着，需要理解的是isHealthy() && hasEnoughSpace()

```java
@Override
public boolean isHealthy() {
    return getMetaFile().exists();
}

@Override
public boolean hasEnoughSpace() {
    return root.getFreeSpace() > freeSpaceMin;
}
```

## 3. RaftStorageMetadata

![1736087167183](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1736087167183.png)

根据Raft需要持久化存储状态的描述，每个Peer需要持久化存储term和votedFor信息，这一点在文章开头的raft-meta文件中也有所体现，整个类比较简单，持有两个实例变量

* long term
* RaftPeerId votedFor

## 4. RaftStorageMetadataFile

接口，两个方法

* getMetadata() 返回RaftStorageMetadata对象
* persit(RaftStorageMetadata newMetadata) 将RaftStorageMetadata对象持久化存储，就是将term和votedFor写入到raft-meta文件中