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

## 5. RaftStorageMetadataFileImpl

相对简单的一个类

* String TERM_KEY = "term"
* String VOTED_FOR_KEY = "votedFor"
* File file
* AtomicReference\<RaftStorageMetadata> metadata

构造器只是对file进行了赋值

```java
  RaftStorageMetadataFileImpl(File file) {
    this.file = file;
  }
```

接口方法

```java
  @Override
  public RaftStorageMetadata getMetadata() throws IOException {
    return ConcurrentUtils.updateAndGet(metadata, value -> value != null? value: load(file));
  }

  @Override
  public void persist(RaftStorageMetadata newMetadata) throws IOException {
    ConcurrentUtils.updateAndGet(metadata,
        old -> Objects.equals(old, newMetadata)? old: atomicWrite(newMetadata, file));
  }
```

这里实际傻瓜对应的是load和store，在getMetadata中，如果metadata已经有值，直接返回，否则调用load(File)从文件中加载；在persist(RaftStorageMetadata)中，如果需要persist的对象为metadata中缓存的对象，直接返回，否则调用atomicWrite(RaftStorageMetadata, File)方法store到磁盘上

```java
  static RaftStorageMetadata load(File file) throws IOException {
    if (!file.exists()) {
      return RaftStorageMetadata.getDefault();
    }
    try(BufferedReader br = new BufferedReader(new InputStreamReader(
        new FileInputStream(file), StandardCharsets.UTF_8))) {
      Properties properties = new Properties();
      properties.load(br);
      return RaftStorageMetadata.valueOf(getTerm(properties), getVotedFor(properties));
    } catch (IOException e) {
      throw new IOException("Failed to load " + file, e);
    }
  }

  static RaftStorageMetadata atomicWrite(RaftStorageMetadata metadata, File file) throws IOException {
    final Properties properties = new Properties();
    properties.setProperty(TERM_KEY, Long.toString(metadata.getTerm()));
    properties.setProperty(VOTED_FOR_KEY, metadata.getVotedFor().toString());

    try(BufferedWriter out = new BufferedWriter(
        new OutputStreamWriter(new AtomicFileOutputStream(file), StandardCharsets.UTF_8))) {
      properties.store(out, "");
    }
    return metadata;
  }
```

## 6. CorruptionPolicy

RaftServerConfigKeys的内部类

```java
    enum CorruptionPolicy {
      /** Rethrow the exception. */
      EXCEPTION,
      /** Print a warn log message and return all uncorrupted log entries up to the corruption. */
      WARN_AND_RETURN;

      public static CorruptionPolicy getDefault() { 
        return EXCEPTION;
      }

      public static <T> CorruptionPolicy get(T supplier, Function<T, CorruptionPolicy> getMethod) {
        return Optional.ofNullable(supplier).map(getMethod).orElse(getDefault());
      }
    }
```

## 7. RaftStorage

在看完RaftStorageDirectory，RaftStorageMetadataFile, CorruptionPolicy后，现在，可以将目光转移到RaftStorage类中

```java
public interface RaftStorage extends Closeable {
  Logger LOG = LoggerFactory.getLogger(RaftStorage.class);

  /** Initialize the storage. */
  void initialize() throws IOException;

  /** @return the storage directory. */
  RaftStorageDirectory getStorageDir();

  /** @return the metadata file. */
  RaftStorageMetadataFile getMetadataFile();

  /** @return the corruption policy for raft log. */
  CorruptionPolicy getLogCorruptionPolicy();

  static Builder newBuilder() {
    return new Builder();
  }

  enum StartupOption {
    /** Format the storage. */
    FORMAT,
    RECOVER
  }

  class Builder {

    private static final Method NEW_RAFT_STORAGE_METHOD = initNewRaftStorageMethod();

    private static Method initNewRaftStorageMethod() {
      final String className = RaftStorage.class.getPackage().getName() + ".StorageImplUtils";
      //final String className = "org.apache.ratis.server.storage.RaftStorageImpl";
      final Class<?>[] argClasses = { File.class, CorruptionPolicy.class, StartupOption.class, long.class };
      try {
        final Class<?> clazz = ReflectionUtils.getClassByName(className);
        return clazz.getMethod("newRaftStorage", argClasses);
      } catch (Exception e) {
        throw new IllegalStateException("Failed to initNewRaftStorageMethod", e);
      }
    }

    private static RaftStorage newRaftStorage(File dir, CorruptionPolicy logCorruptionPolicy,
        StartupOption option, SizeInBytes storageFreeSpaceMin) throws IOException {
      try {
        return (RaftStorage) NEW_RAFT_STORAGE_METHOD.invoke(null,
            dir, logCorruptionPolicy, option, storageFreeSpaceMin.getSize());
      } catch (IllegalAccessException e) {
        throw new IllegalStateException("Failed to build " + dir, e);
      } catch (InvocationTargetException e) {
        Throwable t = e.getTargetException();
        if (t.getCause() instanceof IOException) {
          throw IOUtils.asIOException(t.getCause());
        }
        throw IOUtils.asIOException(e.getCause());
      }
    }


    private File directory;
    private CorruptionPolicy logCorruptionPolicy;
    private StartupOption option;
    private SizeInBytes storageFreeSpaceMin;

    public Builder setDirectory(File directory) {
      this.directory = directory;
      return this;
    }

    public Builder setLogCorruptionPolicy(CorruptionPolicy logCorruptionPolicy) {
      this.logCorruptionPolicy = logCorruptionPolicy;
      return this;
    }

    public Builder setOption(StartupOption option) {
      this.option = option;
      return this;
    }

    public Builder setStorageFreeSpaceMin(SizeInBytes storageFreeSpaceMin) {
      this.storageFreeSpaceMin = storageFreeSpaceMin;
      return this;
    }

    public RaftStorage build() throws IOException {
      return newRaftStorage(directory, logCorruptionPolicy, option, storageFreeSpaceMin);
    }
  }
}
```

