## 1. 说明

类如其名FilePositionCount，记录了三个属性

* File file
* long position
* long count

这里唯一多嘴的就是The class is immutable

```java
public final class FilePositionCount {
  private final File file;
  private final long position;
  private final long count;

  private FilePositionCount(File file, long position, long count) {
    this.file = file;
    this.position = position;
    this.count = count;
  }

  /** @return the file. */
  public File getFile() {
    return file;
  }

  /** @return the starting position. */
  public long getPosition() {
    return position;
  }

  /** @return the byte count. */
  public long getCount() {
    return count;
  }
}
```

