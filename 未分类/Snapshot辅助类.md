## 1. FileInfo

需要记住的有两点

1. value-based class
2. 实例变量 fileSize指的是文件的大小

持有三个实例变量，分别是

* Path pah
* MD5Hash fileDigest
* long fileSize

构造器中对fileSize的赋值是直接取其length()

```java
  public FileInfo(Path path, MD5Hash fileDigest) {
    this.path = path;
    this.fileDigest = fileDigest;
    this.fileSize = path.toFile().length();
  }
```

## 2. FileChunkReader

读取部分文件内容，文件读取的基本单元为chunk，大小为chunkMaxSize和remain中的最小值，值得关注的方法为readFileChunk(int cdhunkMaxSize)方法。根据形参chunkMaxSize和实例变量构造FileChunkProto对象

```java
  public FileChunkProto readFileChunk(int chunkMaxSize) throws IOException {
    final long remaining = info.getFileSize() - offset;
    final int chunkLength = remaining < chunkMaxSize ? (int) remaining : chunkMaxSize;
    final ByteString data = ByteString.readFrom(in, chunkLength);
    final ByteString fileDigest = ByteString.copyFrom(
            digester != null? digester.digest(): info.getFileDigest().getDigest());

    final FileChunkProto proto = FileChunkProto.newBuilder()
        .setFilename(relativePath.toString())
        .setOffset(offset)
        .setChunkIndex(chunkIndex)
        .setDone(offset + chunkLength == info.getFileSize())
        .setData(data)
        .setFileDigest(fileDigest)
        .build();
    chunkIndex++;
    offset += chunkLength;
    return proto;
  }
```

```protobuf
message FileChunkProto {
  string filename = 1; // relative to root
  uint64 totalSize = 2;
  bytes fileDigest = 3;
  uint32 chunkIndex = 4;
  uint64 offset = 5;
  bytes data = 6;
  bool done = 7;
}
```

## 3. InstallSnapshotRequests

Javadoc描述的就很有意思，这里比较难理解的是第一句话，The snaphot is sent by one or more request. 这里认为snapshot是文件格式，可能包含一个或多个文件，每个文件又是以chunk进行划分的，request的单位也是chunk。那么一个snapshot可能会被拆分成一个或者多个请求

```
The snapshot is sent by one or more requests, where a snapshot has one or more files, and a file is sent by one or more chunks.
The number of requests is equal to the sum of the numbers of chunks of each file
```

需要关注的实例变量有两个，分别为

* int fileIndex: snapshot可能包含多个文件，这部分信息在SnapshotInfo中，fileIndex特指snapshotInfo中的fileList的index
* FileChunkReader current: 当前的Reader

重点关注的方法是iterator

```java
class InstallSnapshotRequests implements Iterable<InstallSnapshotRequestProto> {
    @Override
    public Iterator<InstallSnapshotRequestProto> iterator() {
        return new Iterator<InstallSnapshotRequestProto>() {
            @Override
            public boolean hasNext() {
                return fileIndex < snapshot.getFiles().size();
            }
            
            @Override
            public InstallSnapshotRequestProto next() {
                return nextInstallSnapshotRequestProto();
            }
        }
    }
    
  private InstallSnapshotRequestProto nextInstallSnapshotRequestProto() {
    final int numFiles = snapshot.getFiles().size();
    if (fileIndex >= numFiles) {
      throw new NoSuchElementException();
    }
    //遍历snapshot中的fileList，针对每一个file，拆分成chunk大小，构造InstallSnasphotRequestProto对象
    final FileInfo info = snapshot.getFiles().get(fileIndex);
    try {
      if (current == null) {
        current = new FileChunkReader(info, server.getRaftStorage().getStorageDir());
      }
      final FileChunkProto chunk = current.readFileChunk(snapshotChunkMaxSize);
      if (chunk.getDone()) {
        current.close();
        current = null;
        fileIndex++; //只有当前file读取结束后，才会触发fileIndex++，一个文件可能会有多个chunk
      }

      final boolean done = fileIndex == numFiles && chunk.getDone();
      return newInstallSnapshotRequest(chunk, done);
    } catch (IOException e) {
      if (current != null) {
        try {
          current.close();
          current = null;
        } catch (IOException ignored) {
        }
      }
      throw new IllegalStateException("Failed to iterate installSnapshot requests: " + this, e);
    }
  }
}
```

