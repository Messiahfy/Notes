需要注意Glide中、OkHttp中的DiskLruCache有较小的区别，使用时注意一下，但都是来自于JakeWharton的DiskLruCache。

## 主要设计
DiskLruCache 通过在磁盘中创建并维护一个Journal文件来记录各种缓存操作，供初始化时生成 LinkedHashMap 用，也就是LRU的持久化。同时例如其中的DIRTY记录也可以用于判断正在编辑的文件，如果后面没有同一个key的CLEAN或者REMOVE记录，说明未修改成功就结束了流程，例如异常、断电等情况。

在修改文件时，都会使用备份文件这种思路，避免修改过程异常


前四行：魔数、DiskLruCache的版本、app的版本、valueCount。
```
libcore.io.DiskLruCache
1
100
2

CLEAN 3400330d1dfc7f3f7f4b8d4d803dfcf6 832 21054
DIRTY 335c4c6028171cfddfbaae1a9c313c52
CLEAN 335c4c6028171cfddfbaae1a9c313c52 3934 2342
REMOVE 335c4c6028171cfddfbaae1a9c313c52
DIRTY 1ab96a171faeeee38496d8b330771a7a
CLEAN 1ab96a171faeeee38496d8b330771a7a 1600 234
READ 335c4c6028171cfddfbaae1a9c313c52
READ 3400330d1dfc7f3f7f4b8d4d803dfcf6
```

1. DIRTY表示进入编辑状态，待写入，会在diskLruCache.edit(key)时创建，并没有真正的缓存。后面必然会跟一个CLEAN或者REMOVE，否则视为错误的临时文件需要删除
2. CLEAN表示数据已经成功写入，其后还跟了当前数据的大小，valueCount大于1，则有多个数据。
3. READ表示当前数据被get一次，用于更新Lru顺序。
4. REMOVE表示当前数据已经被移除。


## 使用情况

1. 首次创建，open方法中只会执行最后几步，创建目录和DiskLruCache对象，并且调用rebuildJournal()，
2. 因为是首次创建，rebuildJournal()中实际就是写好journalFileTmp文件，然后把它重命名为journal，并创建对应的BufferedWriter，
3. 要写入数据，调用edit方法，因为是首次创建，所以会创建新的Entry，每个Entry都对应一个key，和以key命名的文件和tmp文件，根据valueCount情况，例如“$key.0”、“$key.1”、“$key.0.tmp”、“$key.1.tmp”。Entry会存到lruEntries中
4. 调用edit会得到一个Editor对象，然后就可以调用例如getFile(index)对文件或数据流操作，修改数据。getFile(index)会把Editor的written[index] 设为 true，并且返回temp文件用户写文件，例如“$key.0.tmp”
5. 写入数据后，调用commit()或者abort()，commit后会把修改的临时文件重命名为正式文件，并记录CLEAN到journal文件中，abort则删除临时文件，并记录
6. 读取数据，调用get


journal备份文件，在rebuildJournal()重建journal文件时会创建它，修改成功后将备份文件重命名为正式journal文件，可以避免修改journal文件过程中发生异常、断电等情况，导致记录丢失。


再次open的情况，readJournal()会从journal文件中读取数据，遍历CLEAN、DIRTY、READ、REMOVE这些数据，构造entry存入lruEntries中。比如某个key对应的记录最后是REMOVE，那么这个entry不会存在于lruEntries中；比如READ会影响entry在lruEntries中的位置，用于LRU。


## 源码分析
```
public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)
      throws IOException {

    // 参数检查
    if (maxSize <= 0) {
      throw new IllegalArgumentException("maxSize <= 0");
    }
    if (valueCount <= 0) {
      throw new IllegalArgumentException("valueCount <= 0");
    }


    File backupFile = new File(directory, JOURNAL_FILE_BACKUP);
    // 如果备份文件存在，就使用它
    if (backupFile.exists()) {
      File journalFile = new File(directory, JOURNAL_FILE);
      // 如果journal文件存在，则删除备份文件
      if (journalFile.exists()) {
        backupFile.delete();
      } else {
        // 否则，把备份文件重命名为journal文件
        renameTo(backupFile, journalFile, false);
      }
    }

    DiskLruCache cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
    // 如果journal文件存在，比如已经记录了数据，关闭app重新打开并调用open
    if (cache.journalFile.exists()) {
      try {
        // 从journal读取数据，存入lruEntries，也就是通过journal文件创建内存中的lru记录
        cache.readJournal();
        cache.processJournal();
        return cache;
      } catch (IOException journalIsCorrupt) {
        System.out
            .println("DiskLruCache "
                + directory
                + " is corrupt: "
                + journalIsCorrupt.getMessage()
                + ", removing");
        cache.delete();
      }
    }

    // 没有journal文件，或者版本变化，需要重新构造新的journal文件
    directory.mkdirs();
    cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
    cache.rebuildJournal();
    return cache;
  }
```

LinkedHashMap会按顺序放置数据，后put的放到后面，所以遍历的时候就是按写入顺序得到数据。accessOrder为true的话，get数据会让访问的数据放到最后。如果充写removeEldestEntry的话，可以实现内存LRU。当然，这里并没有重写，因为并不需要自动移除旧数据，只需要把最新访问的数据放到最后也就是最新的位置即可。

rebuildJournal，通过内存中的lruEntries，创建一个新的日志，省略冗余信息。这将替换当前日志（如果存在）
```
private synchronized void rebuildJournal() throws IOException {
    // 如果存在journal文件的Writer，先关掉它
    if (journalWriter != null) {
      closeWriter(journalWriter);
    }

    // 创建新的Writer，但这个Writer是对于journalFileTmp
    Writer writer = new BufferedWriter(
        new OutputStreamWriter(new FileOutputStream(journalFileTmp), Util.US_ASCII));
    try {
      // 写入文件开头：魔数、DiskLruCache版本、app版本、valueCount（每个key可以对应多少文件）
      writer.write(MAGIC);
      writer.write("\n");
      writer.write(VERSION_1);
      writer.write("\n");
      writer.write(Integer.toString(appVersion));
      writer.write("\n");
      writer.write(Integer.toString(valueCount));
      writer.write("\n");
      writer.write("\n");

      // 将lruEntries中的记录数据写入journalFileTmp，初始情况为空，使用过程中调用的话会有数据
      for (Entry entry : lruEntries.values()) {
        if (entry.currentEditor != null) {
          // 
          writer.write(DIRTY + ' ' + entry.key + '\n');
        } else {
          // 
          writer.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
        }
      }
    } finally {
      closeWriter(writer);
    }

    // 如果journal文件存在，删除原本journalFileBackup（如果存在），并将journal文件重命名为journalFileBackup
    // 也就是修改journal前，先备份一下，出现异常的话，还可以通过备份文件恢复
    if (journalFile.exists()) {
      renameTo(journalFile, journalFileBackup, true);
    }
    // 然后把journalFileTmp重命名为journal文件
    renameTo(journalFileTmp, journalFile, false);
    // 重命名为journal文件成功，所以删除备份文件
    journalFileBackup.delete();

    // 对journal文件创建对应的BufferedWriter
    journalWriter = new BufferedWriter(
        new OutputStreamWriter(new FileOutputStream(journalFile, true), Util.US_ASCII));
}
```



readJournal函数，仅在open时调用：
```
private void readJournal() throws IOException {
    // 这个StrictLineReader内部在读到文件尾部还是没有换行符的话，通过hasUnterminatedLine可以判断这种情况
    StrictLineReader reader = new StrictLineReader(new FileInputStream(journalFile), Util.US_ASCII);
    try {
      // 如果魔数、版本或者valueCount变化，或者没有空格行，则抛出异常，
      // 在上面介绍的open方法可以看到会catch并删除缓存，然后重建新的Journal文件，
      String magic = reader.readLine();
      String version = reader.readLine();
      String appVersionString = reader.readLine();
      String valueCountString = reader.readLine();
      String blank = reader.readLine();
      if (!MAGIC.equals(magic)
          || !VERSION_1.equals(version)
          || !Integer.toString(appVersion).equals(appVersionString)
          || !Integer.toString(valueCount).equals(valueCountString)
          || !"".equals(blank)) {
        throw new IOException("unexpected journal header: [" + magic + ", " + version + ", "
            + valueCountString + ", " + blank + "]");
      }

      // 读取Journal文件中除开头的固定格式外的剩余操作记录数据
      int lineCount = 0;
      while (true) {
        try {
          // 循环调用readJournalLine，最终就会把journal中的记录，根据情况把各个entry存到lruEntries中
          readJournalLine(reader.readLine());
          lineCount++;
        } catch (EOFException endOfJournal) {
          // 读到结束会抛出EOFException，就中止循环
          break;
        }
      }
      // redundantOpCount为记录行数减去lruEntries的数量，也就是操作记录数量减去实际存在的entry的数量
      redundantOpCount = lineCount - lruEntries.size();

      // 如果从某个位置开始，读完文件还是没有换行符，就重建journal文件
      if (reader.hasUnterminatedLine()) {
        rebuildJournal();
      } else {
        // 创建新的journalWriter
        journalWriter = new BufferedWriter(new OutputStreamWriter(
            new FileOutputStream(journalFile, true), Util.US_ASCII));
      }
    } finally {
      Util.closeQuietly(reader);
    }
  }
```

读取Journal中的一行操作记录，例如“CLEAN 3400330d1dfc7f3f7f4b8d4d803dfcf6”。

* 读到REMOVE：会把lruEntries的对应记录删除
* 不是REMOVE，如果Entry不存在，都会创建对应的Entry并放到lruEntries中
* 读到CLEAN：把entry的readable设为true，currentEditor设为null，并记录文件字节长度数据
* 读到DIRTY：创建Editor赋值给entry的currentEditor
* 读到READ：因为调用了lruEntries.get(key)，所以如果lruEntries存在该数据
```
  private void readJournalLine(String line) throws IOException {
    // 找到第一个空格位置，也就是CLEAN、READ等操作字符串和key的间隔
    int firstSpace = line.indexOf(' ');
    if (firstSpace == -1) {
      throw new IOException("unexpected journal line: " + line);
    }

    // key开始的位置，即空格的下一个位置
    int keyBegin = firstSpace + 1;
    // 寻找第二个空格（比如CLEAN的情况，key后面隔一个空格还会有文件的字节数量）
    int secondSpace = line.indexOf(' ', keyBegin);
    final String key;
    if (secondSpace == -1) {
      // 如果只有一个空格
      key = line.substring(keyBegin);
      // 这一行是REMOVE记录，就从lruEntries中删除该数据。应该是对应非初始情况
      if (firstSpace == REMOVE.length() && line.startsWith(REMOVE)) {
        lruEntries.remove(key);
        // 结束
        return;
      }
    } else {
      // 如果有两个空格，仍然拿到key
      key = line.substring(keyBegin, secondSpace);
    }

    // 根据journal文件记录的数据，创建内存中的Entry，存入lruEntries
    Entry entry = lruEntries.get(key);
    if (entry == null) {
      entry = new Entry(key);
      lruEntries.put(key, entry);
    }

    if (secondSpace != -1 && firstSpace == CLEAN.length() && line.startsWith(CLEAN)) {
      // 如果是CLEAN记录，并且有字节数量的数据，就取出字节数量这部分数据
      String[] parts = line.substring(secondSpace + 1).split(" ");
      // 因为是CLEAN数据，所以设置readable为true
      entry.readable = true;
      entry.currentEditor = null;
      // 赋值文件的长度数据
      entry.setLengths(parts);
    } else if (secondSpace == -1 && firstSpace == DIRTY.length() && line.startsWith(DIRTY)) {
      // DIRTY数据，
      entry.currentEditor = new Editor(entry);
    } else if (secondSpace == -1 && firstSpace == READ.length() && line.startsWith(READ)) {
      // READ数据，因为前面调用lruEntries.get()，所以该数据会放到最新位置，所以这里不用操作
    } else {
      // 异常数据 
      throw new IOException("unexpected journal line: " + line);
    }
  }
```

计算总文件大小，并且把DIRTY数据删除掉。
```
  private void processJournal() throws IOException {
    deleteIfExists(journalFileTmp);
    for (Iterator<Entry> i = lruEntries.values().iterator(); i.hasNext(); ) {
      Entry entry = i.next();
      if (entry.currentEditor == null) {
        for (int t = 0; t < valueCount; t++) {
          size += entry.lengths[t];
        }
      } else {
        // readJournalLine会对DIRTY数据对应的entry的currentEditor赋值一个Editor，如果后面跟着有CLEAN，
        // 则这个currentEditor会为null，或者后面有REMOVE，则entry不会存在。所以这种情况就是只有DIRTY，
        // 比如调用edit却没有调用commit或者abort，例如断电或其他异常情况。就直接删除该缓存
        entry.currentEditor = null;
        for (int t = 0; t < valueCount; t++) {
          deleteIfExists(entry.getCleanFile(t));
          deleteIfExists(entry.getDirtyFile(t));
        }
        i.remove();
      }
    }
  }
```

编辑数据
```
public Editor edit(String key) throws IOException {
    return edit(key, ANY_SEQUENCE_NUMBER);
}

private synchronized Editor edit(String key, long expectedSequenceNumber) throws IOException {
    // 检查journalWriter是否存在。如果调用close就会置空
    checkNotClosed();
    // 通过key获取Entry
    Entry entry = lruEntries.get(key);
    
    // expectedSequenceNumber != ANY_SEQUENCE_NUMBER 对应通过get(index)的Value调用edit的情况：
    // 这种情况如果entry为null（被删除）或者sequenceNumber!= expectedSequenceNumber（比如另一个调用edit后commit），
    // 就直接返回null，因为Value是快照，这种情况Value已经被修改，不能使用
    if (expectedSequenceNumber != ANY_SEQUENCE_NUMBER && (entry == null
        || entry.sequenceNumber != expectedSequenceNumber)) {
      return null; // Value is stale.
    }
    if (entry == null) {
      // key对应的文件不存在，创建新的Entry
      entry = new Entry(key);
      // 并存到lruEntries中
      lruEntries.put(key, entry);
    } else if (entry.currentEditor != null) {
      // 已经调用edit，并且没有commit/abort，也就是正常编辑中，无法并发操作
      return null; // Another edit is in progress.
    }

    // 创建一个Editor
    Editor editor = new Editor(entry);
    // 赋值entry的currentEditor
    entry.currentEditor = editor;

    // 把操作记录写入journal文件，因为这里要开始编辑，所有标记为DIRTY，假如DIRTY只会没有CLEAN或者REMOVE，
    // 则DIRTY的数据在重新open的时候会被删除
    journalWriter.append(DIRTY);
    journalWriter.append(' ');
    journalWriter.append(key);
    journalWriter.append('\n');
    // flush确保写入
    flushWriter(journalWriter);
    return editor;
}
```


Entry代表一个key对应的所有数据的记录
```
private final class Entry {
    private final String key;

    // 当前key对应的若干文件，每个文件对应的字节数
    private final long[] lengths;

    // 对应"key.0"、"key.1"，根据valueCount决定
    File[] cleanFiles;
    // 对应"key.0.tmp"、"key.1.emp"
    File[] dirtyFiles;

    // 只要成功写入一次当前entry的key对应的文件数据（即commit），readable就会设为true，后续的修改不影响
    private boolean readable;

    // 正在编辑当前key对应数据时，会创建对应的Editor
    private Editor currentEditor;

    // 序号，每次commit会+1，用于判断数据是否已经被修改
    private long sequenceNumber;

    // 构造Entry，根据valueCount，创建多个数组
    private Entry(String key) {
      this.key = key;
      this.lengths = new long[valueCount];
      cleanFiles = new File[valueCount];
      dirtyFiles = new File[valueCount];

      // cleanFiles数组为实际的数据文件，dirtyFiles则为tmp文件
      StringBuilder fileBuilder = new StringBuilder(key).append('.');
      int truncateTo = fileBuilder.length();
      for (int i = 0; i < valueCount; i++) {
          fileBuilder.append(i);
          // 创建名为key.i的文件，比如：232dsadsadsd.0
          cleanFiles[i] = new File(directory, fileBuilder.toString());
          fileBuilder.append(".tmp");
          // 创建名为key.i.tmp的文件，比如：232dsadsadsd.0.tmp
          dirtyFiles[i] = new File(directory, fileBuilder.toString());
          // 重置为“key.”
          fileBuilder.setLength(truncateTo);
      }
    }

    // ......
  }
```

Entry对应的Editor，修改的过程也就是调用edit后，Entry的currentEditor存在，commit/abort后为null
```
  public final class Editor {
    private final Entry entry;
    private final boolean[] written;
    private boolean committed;

    private Editor(Entry entry) {
      this.entry = entry;
      // entry还没有写入过数据的情况，readable为false，此时written为new boolean[valueCount]，否则为null
      this.written = (entry.readable) ? null : new boolean[valueCount];
    }

    /**
     * Returns an unbuffered input stream to read the last committed value,
     * or null if no value has been committed.
     */
    private InputStream newInputStream(int index) throws IOException {
      synchronized (DiskLruCache.this) {
        if (entry.currentEditor != this) {
          throw new IllegalStateException();
        }
        if (!entry.readable) {
          return null;
        }
        try {
          return new FileInputStream(entry.getCleanFile(index));
        } catch (FileNotFoundException e) {
          return null;
        }
      }
    }

    // .....

    public File getFile(int index) throws IOException {
      synchronized (DiskLruCache.this) {
        if (entry.currentEditor != this) {
            throw new IllegalStateException();
        }
        if (!entry.readable) {
            written[index] = true;
        }
        File dirtyFile = entry.getDirtyFile(index);
        directory.mkdirs();
        return dirtyFile;
      }
    }

    /**
     * Commits this edit so it is visible to readers.  This releases the
     * edit lock so another edit may be started on the same key.
     */
    public void commit() throws IOException {
      // The object using this Editor must catch and handle any errors
      // during the write. If there is an error and they call commit
      // anyway, we will assume whatever they managed to write was valid.
      // Normally they should call abort.
      completeEdit(this, true);
      // editor提交后，committed设为true
      committed = true;
    }

    // 中止（取消）编辑。
    public void abort() throws IOException {
      completeEdit(this, false);
    }
  }
```

提交或者中止
```
private synchronized void completeEdit(Editor editor, boolean success) throws IOException {
    Entry entry = editor.entry;
    if (entry.currentEditor != editor) {
      throw new IllegalStateException();
    }

    // entry.readable为false，说明还没有写入过数据，此时commit
    if (success && !entry.readable) {
      for (int i = 0; i < valueCount; i++) {
        // Editor的getFile方法对于readable为false的情况，会赋值written[index]为true，如果某个没有为true，会抛出异常。
        // 也就是说，valueCount > 1的时候，每次编辑，每个索引都要写入数据，即调用每个index的getFile
        if (!editor.written[i]) {
          editor.abort();
          throw new IllegalStateException("Newly created entry didn't create value for index " + i);
        }
        // Editor的getFile方法中会创建temp文件，这里就是检测一下，不存在则不能提交
        if (!entry.getDirtyFile(i).exists()) {
          editor.abort();
          return;
        }
      }
    }

    for (int i = 0; i < valueCount; i++) {
      File dirty = entry.getDirtyFile(i);
      if (success) {
        // commit情况，会把temp重命名为正式的数据文件
        if (dirty.exists()) {
          File clean = entry.getCleanFile(i);
          dirty.renameTo(clean);
          long oldLength = entry.lengths[i];
          long newLength = clean.length();
          // 更新新的数据的字节长度
          entry.lengths[i] = newLength;
          // 得到新的size
          size = size - oldLength + newLength;
        }
      } else {
        // false情况，即abort，则删除dirty临时文件，不修改正式的文件
        deleteIfExists(dirty);
      }
    }

    redundantOpCount++;
    // 编辑结束，currentEditor设为null
    entry.currentEditor = null;
    if (entry.readable | success) { // 曾经写入成功过，或者本次commit
      entry.readable = true;
      // 添加CLEAN记录
      journalWriter.append(CLEAN);
      journalWriter.append(' ');
      journalWriter.append(entry.key);
      journalWriter.append(entry.getLengths());
      journalWriter.append('\n');

      if (success) {
        // 本次写入成功，序号+1
        entry.sequenceNumber = nextSequenceNumber++;
      }
    } else {
      // 首次写入，失败，删除entry，并写入REMOVE记录（跟在DIRTY后面）
      lruEntries.remove(entry.key);
      journalWriter.append(REMOVE);
      journalWriter.append(' ');
      journalWriter.append(entry.key);
      journalWriter.append('\n');
    }
    flushWriter(journalWriter);

    // 如果总大小超过限制，或者redundantOpCount超过2000并且redundantOpCount超过lruEntries的size，
    // 也就是说redundantOpCount超过2000，但没有超过lruEntries的话，不用执行清理操作
    if (size > maxSize || journalRebuildRequired()) {
      executorService.submit(cleanupCallable);
    }
}
```

```
  public synchronized Value get(String key) throws IOException {
    checkNotClosed();
    // 调用get，如果数据存在，就会在lruEntries中，把这个key的数据移到最新位置
    Entry entry = lruEntries.get(key);
    if (entry == null) {
      return null;
    }

    if (!entry.readable) {
      return null;
    }

    for (File file : entry.cleanFiles) {
        // A file must have been deleted manually!
        if (!file.exists()) {
            return null;
        }
    }

    redundantOpCount++;
    // 记录READ操作到journal文件
    journalWriter.append(READ);
    journalWriter.append(' ');
    journalWriter.append(key);
    journalWriter.append('\n');
    if (journalRebuildRequired()) {
      executorService.submit(cleanupCallable);
    }

    // 返回Value，可以通过Value读取数据
    return new Value(key, entry.sequenceNumber, entry.cleanFiles, entry.lengths);
  }
```

// 超过size限制的情况，删除LRU最久未操作的文件；READ、CLEAN、REMOVE都会让redundantOpCount+1，
// redundantOpCount超过2000并且超过lruEntries的size，就需要重新构建journal文件，也就是不让journal文件太大，把多余的记录删掉。
```
private final Callable<Void> cleanupCallable = new Callable<Void>() {
    public Void call() throws Exception {
      synchronized (DiskLruCache.this) {
        if (journalWriter == null) {
          return null; // Closed.
        }
        trimToSize(); // 删除旧数据，直到小于文件大小限制。会删除lruEntries的记录，并在journal文件中记录REMOVE
        if (journalRebuildRequired()) {
          // 读取lruEntries，根据记录的所有entry，重新写journal文件，也就是把没有用的记录删掉。
          rebuildJournal();
          redundantOpCount = 0;
        }
      }
      return null;
    }
};

// 如果超过大小限制，删除最旧的数据，直到小于限制
private void trimToSize() throws IOException {
    while (size > maxSize) {
      Map.Entry<String, Entry> toEvict = lruEntries.entrySet().iterator().next();
      remove(toEvict.getKey());
    }
}

// 是否需要重新创建journal文件。在多余的操作数redundantOpCount超过2000并且超过lruEntries的大小时，才需要重新创建
private boolean journalRebuildRequired() {
    final int redundantOpCompactThreshold = 2000;
    return redundantOpCount >= redundantOpCompactThreshold //
        && redundantOpCount >= lruEntries.size();
  }
```