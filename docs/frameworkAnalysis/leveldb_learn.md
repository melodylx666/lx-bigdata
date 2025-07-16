# LevelDB源码学习

单机KV存储的入门经典，LSM-Tree Based

## 文章
知乎上的文章已经十分丰富,下面的系列已经非常完备:
[LevelDB源码解读](https://www.zhihu.com/column/c_1258068131073183744)


## src
这里选择Java 版本的[leveldb](https://github.com/dain/leveldb)

跟着单元测试打断点走读就可以了，入手最快的方式就是跟着单测debug(当单测完备的情况下)


## 代码组织
1. leveldb-api：用于定义leveldb的公共=接口和抽象类。
2. leveldb，包含leveldb-java的完整实现，包括数据库的读写，压缩，快照。
3. leveldb-benchmark，进行性能基准测试。

## leveldb-api
### 核心接口
#### DB
使用Map.Entry作为元组表示键值对，使用字节数组作为Key & Value。
1. 支持查询：get
2. 支持单数据模型的 put,delete,write,  无返回值
3. 支持多数据模型的 put,delete,write，返回值为经过该操作之后的snapshot
4. 支持直接返回DB的快照
5. 支持返回增强迭代器(DBIterator)，用于遍历数据。
```java
public interface DB
        extends Iterable<Map.Entry<byte[], byte[]>>, Closeable
{
    byte[] get(byte[] key)
            throws DBException;

    byte[] get(byte[] key, ReadOptions options)
            throws DBException;

    @Override
    DBIterator iterator();

    DBIterator iterator(ReadOptions options);

    void put(byte[] key, byte[] value)
            throws DBException;

    void delete(byte[] key)
            throws DBException;

    void write(WriteBatch updates)
            throws DBException;

    WriteBatch createWriteBatch();

    /**
     * @return null if options.isSnapshot()==false otherwise returns a snapshot
     * of the DB after this operation.
     */
    Snapshot put(byte[] key, byte[] value, WriteOptions options)
            throws DBException;

    /**
     * @return null if options.isSnapshot()==false otherwise returns a snapshot
     * of the DB after this operation.
     */
    Snapshot delete(byte[] key, WriteOptions options)
            throws DBException;

    /**
     * @return null if options.isSnapshot()==false otherwise returns a snapshot
     * of the DB after this operation.
     */
    Snapshot write(WriteBatch updates, WriteOptions options)
            throws DBException;

    Snapshot getSnapshot();
}
```
#### DBIterator
实现了Iterator，并额外增加了功能
1. seek: 将迭代位置定位到大于等于某个key的位置
2. seekToFirst：将迭代位置定位到开头
3. peekNext：得到下个数据，但是不移动迭代位置

```java
public interface DBIterator
        extends Iterator<Map.Entry<byte[], byte[]>>, Closeable
{
    /**
     * Repositions the iterator so the key of the next BlockElement
     * returned greater than or equal to the specified targetKey.
     */
    void seek(byte[] key);

    /**
     * Repositions the iterator so is is at the beginning of the Database.
     */
    void seekToFirst();

    /**
     * Returns the next element in the iteration, without advancing the iteration.
     */
    Map.Entry<byte[], byte[]> peekNext();

}
```

#### DBFactory
工厂模式接口，用于创建和管理DB实例
```java
public interface DBFactory
{
    //基于path & options，创建DB实例
    DB open(File path, Options options)
            throws IOException;
    //基于path & options，销毁DB实例
    void destroy(File path, Options options)
            throws IOException;
}
```
### 配置类
#### Options
数据库配置类
1. 是否自动创建数据库文件
2. 写缓冲区(writeBuffer)大小，缓存(cache)大小
3. 压缩类型，默认Snappy。只有Snappy和None两种。
4. 其他

#### ReadOptions
1. 是否填充缓存cache，默认是true
2. 是否验证读取数据的checksum
3. 是否指定并返回快照
使用方式，支持流式和常规两种创建方式。
```java
Options options = new Options();
options.compressionType(CompressionType.NONE);

//或者
Options options = new Options().compressionType(CompressionType.NONE);
```
#### WriteOptions
1. 控制sync/async落盘
2. 是否返回快照
和ReadOptions使用方式完全一致。

### 其他
#### SnapShot
作为数据库在某个时间点的快照视图，需要手动释放资源。
public interface Snapshot
        extends Closeable
{
}
#### DBCompartor
作用：用于自定义键排序策略。
DBException
继承自RuntimeException，便于上层调用者捕获处理。

## leveldb
### 功能测试
#### ApiTest
功能：用于验证整体功能的交互是否满足。通过典型的写入 -> 删除 ->写入循环，验证是否在多次compaction之后依旧满足要求。
核心逻辑解读：
1. 创建本次DB存储path："/home/xxx/leveldb-099901/testCompaction"
2. 通过path & option，创建DB
3. 循环测试，
  1. 写入：将100万个key-value，转换为bytes，写入。
  2. 关闭DB connection，调用的是DBImpl的close方法，DB数据不会删除
  3. 新建DB connection，基于同样的路径，这里会重新创建:new DbImpl(options, path)。
  4. 删除：将100万个key-value删除
  5. 关闭DB connection
### 核心实现类
#### DbImpl
写操作.
写操作完全串行化，属于单线程写
1. 输入参数
   1. updates：一个写操作批次，包含多个put/delete
   2. options：WriteOptions，上面我们分析过，是控制是否sync落盘，以及是否返回快照
2. 核心逻辑
   1. 通过变量检查后台线程是否异常
   2. 加上独占锁
   3.  如果写请求 > 0
       1.  检查memtable是否需要更新，这里下面详细解析
       2.  得到当前updates的起止序列号
       3.   更新全局版本终止序列号
       4.   将updates操作序列化为Slice形式的WAL日志，同步/异步持久化到磁盘
       5.   将updates的每个操作执行，更新memtable.
    4.   如果写请求 == 0. 得到当前空操作序列的起止序列号
    5.   根据操作序列的起止序列号，返回应用完当前操作的快照(包Version & lastseqEnd)
    6.   释放独占锁

```java
public Snapshot writeInternal(WriteBatchImpl updates, WriteOptions options)
        throws DBException
{
    //整个DBImpl维护一个e变量，代表后台compation线程是否异常
    checkBackgroundException();
    //独占锁，这里用ReentrantLock
    mutex.lock();
    try {
        long sequenceEnd;
        //如果有写请求
        if (updates.size() != 0) {
            //保证写请求在内存中的写入空间大小，可能触发memo table切换，将旧memo table变immutable
            makeRoomForWrite(false);
            
            // Get sequence numbers for this change set
            //leveldb使用全局递增的序列号来保持操作顺序和快照一致性
            long sequenceBegin = versions.getLastSequence() + 1;
            //这一批写入使用连续的序列号
            sequenceEnd = sequenceBegin + updates.size() - 1;
            //更新全局versions的最新最后一个序列号
            versions.setLastSequence(sequenceEnd);
            
            // Log write。此处将整个WriteBatch序列化为一条WAL日志(字节数组)，用于恢复
            //Slice是封装好的byte array
            Slice record = writeWriteBatch(updates, sequenceBegin);
            try {
                //DBImpl维护一个logger writer，用于写WAL日志。如果sync为true，则强制刷盘
                //记住WAL日志写的就是key-value序列化的值，以及对应的操作信息。
                log.addRecord(record, options.sync());
            }
            catch (IOException e) {
                throw Throwables.propagate(e);
            }

            // Update memtable。遍历所有写操作(seqBegin-seqEnd)，将记录加入到memTable中
            updates.forEach(new InsertIntoHandler(memTable, sequenceBegin));
        }
        else {
            //如果实际没有数据写入，则当前写组合的seq_end就是versions.last
            sequenceEnd = versions.getLastSequence();
        }

        if (options.snapshot()) {
            //要返回当前写操作之后的快照状态
            //这里封装的是两个信息:当前Version，以及是从0到哪个位置的状态叠加:sequenceEnd，屏蔽掉之后的修改
            return new SnapshotImpl(versions.getCurrent(), sequenceEnd);
        }
        else {
            return null;
        }·
    }
    finally {
        //释放独占锁
        mutex.unlock();
    }
}
```

其中的makeRoomForWrite如下：
```java

    private void makeRoomForWrite(boolean force)
    {
        //检查当前线程是否持有独占锁来操作memotable
        checkState(mutex.isHeldByCurrentThread());
        
        boolean allowDelay = !force;

        while (true) {
            //case1：如果需要压缩，并且允许延迟，则释放锁并sleep1s，让出cpu使得memtable被快速压缩
            if (allowDelay && versions.numberOfFilesInLevel(0) > L0_SLOWDOWN_WRITES_TRIGGER) {
                try {
                    mutex.unlock();
                    Thread.sleep(1);
                }
                catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException(e);
                }
                finally {
                    mutex.lock();
                }

                // Do not delay a single write more than once
                allowDelay = false;
            }
            //case2：当前的memtable都没满
            else if (!force && memTable.approximateMemoryUsage() <= options.writeBufferSize()) {
                // There is room in current memtable
                break;
            }
            //case3:immutable memtable还没有压缩释放
            else if (immutableMemTable != null) {
                // We have filled up the current memtable, but the previous
                // one is still being compacted, so we wait.
                backgroundCondition.awaitUninterruptibly();
            }
            //case4:迫切需要压缩，则当前线程处于阻塞在条件队列中，等待其他线程将其唤醒。
            else if (versions.numberOfFilesInLevel(0) >= L0_STOP_WRITES_TRIGGER) {
                // There are too many level-0 files.
//                Log(options_.info_log, "waiting...\n");
                backgroundCondition.awaitUninterruptibly();
            }
            else {
            //case5:创建一个新的memtable，将旧的进行压缩
                // Attempt to switch to a new memtable and trigger compaction of old
                checkState(versions.getPrevLogNumber() == 0);

                // close the existing log
                try {
                    log.close();
                }
                catch (IOException e) {
                    throw new RuntimeException("Unable to close log file " + log.getFile(), e);
                }

                // open a new log
                long logNumber = versions.getNextFileNumber();
                try {
                    this.log = Logs.createLogWriter(new File(databaseDir, Filename.logFileName(logNumber)), logNumber);
                }
                catch (IOException e) {
                    throw new RuntimeException("Unable to open new log file " +
                            new File(databaseDir, Filename.logFileName(logNumber)).getAbsoluteFile(), e);
                }

                // create a new mem table
                immutableMemTable = memTable;
                memTable = new MemTable(internalKeyComparator);

                // Do not force another compaction there is space available
                force = false;

                maybeScheduleCompaction();
            }
        }
    }
```

读操作。
读操作在SST Table搜索的时候可以并发，而在memtable中是串行化。所以它可以支持多线程并发读取。
ReadOptions功能如下：
1. 是否填充缓存cache，默认是true
2. 是否验证读取数据的checksum
3. 是否指定并返回快照实现
读取的byte[] key对应的value,比如key是[98,97,114]

```java
public byte[] get(byte[] key, ReadOptions options)
        throws DBException
{
    //检查后台线程异常
    checkBackgroundException();
    
    LookupKey lookupKey;
    //加上独占锁，操作memtable,以及immutable table
    mutex.lock();
    try {
        //根据options返回指定快照，或者创建最新快照
        SnapshotImpl snapshot = getSnapshot(options);
        //基于原始key，以及当前的lastSeq number，创建用于查找的key
        lookupKey = new LookupKey(Slices.wrappedBuffer(key), snapshot.getLastSequence());

        // First look in the memtable, then in the immutable memtable (if any).
        
        //一个DbImpl分别维护一个memtable，以及一个immutable table
        //首先在memo table中查看是否有对应key对应的value
        LookupResult lookupResult = memTable.get(lookupKey);
        if (lookupResult != null) {
            Slice value = lookupResult.getValue();
            if (value == null) {
                return null;
            }
            return value.getBytes();
        }
        //然后在immutable table中查看是否有
        if (immutableMemTable != null) {
            lookupResult = immutableMemTable.get(lookupKey);
            if (lookupResult != null) {
                Slice value = lookupResult.getValue();
                if (value == null) {
                    return null;
                }
                return value.getBytes();
            }
        }
    }
    finally {
        //释放独占锁，允许其他操作访问table
        mutex.unlock();
    }
    
    //在SST文件(磁盘中查找)，也就是调用versions.get(lookupkey)
    //这里会合并迭代器数据，将数据截止到lastVersionEnd的最新版本合并出来
    // Not in memTables; try live files in level order
    LookupResult lookupResult = versions.get(lookupKey);

    // schedule compaction if necessary
    //在这里触发SST文件压缩，将压缩任务提交到DbImpl维护的线程池中执行
    //为什么要在读取的时候触发压缩，而不是写入的时候？
    //1. 避免阻塞写入
    //2. 减少后续查询开销
    mutex.lock();
    try {
        if (versions.needsCompaction()) {
            maybeScheduleCompaction();
        }
    }
    finally {
        mutex.unlock();
    }
    
    //返回结果
    if (lookupResult != null) {
        Slice value = lookupResult.getValue();
        if (value != null) {
            return value.getBytes();
        }
    }
    return null;
}
```

上面只有读写操作的分析，下面我们来看其他部分。

#### SnapShotImpl

SnapsShotImpl是快照实现类，用于返回指定快照。本质上它是Version & SequenceNumber的封装。

#### Version & VersionSet
管理并维护所有活跃的Version实例，同时负责apply VersionEdit以产生新的Version实例。

一个Version代表一个具体的SSTable文件集合的状态(包括Level0以及其他层级)。当引用计数=0的时候，会被从VersionSet中删除掉。

#### VersionEdit & Manifest file

是一种中间数据结构，用于表示1次对Version的更改(也就是对SSTable文件的修改)。而Manifest file则是一个log文件，每一个log record就是一个VersionEdit序列化为Slice之后的结果。

那么非常显然，Version是VersionSet叠加之后的数据库状态，包含当前的SSTable文件状态。也就是说，其实通过Manifest file,就可以知道则合格那个LevelDB的状态。

#### Compaction
这个不是一个类，而是一个核心过程。在写操作开始的时候会通过makeRoomForWrite()，有可能触发Compaction。而读操作也会在读取完毕之后手动触发Compaction。

Compaction会带来严重的写放大问题，也就是写入一个record的时机IO写入(所有因此产生的写操作)流量会是理论的数倍。这是由于WriteInternal的时候，首先会写WAL(包含完整的record以及op)，然后从immutable memTable到Level-0落盘，再到level-0与level-1的合并，再到level-n与level-n+1的合并(写入量上升一个数量级)。总共会产生大一个数量级以上的IO写入流量。

详细来看，假设一次写入n个record：

- WAL，会是1.x n
- immutable memTable落盘为level-0，会是n
- level0和level1的compact，这里是读取level-0的所有record，和level-1的对应范围的Record进行归并排序。假设原始写入一个memTable是2MB,但是现在需要将level-0(4 * 2)和level-1的文件(1 * 10)合并再写入，则为18MB,经过压缩过滤，实际为2.x倍。
- level-n到level-n+1的compact，这里数据倍数关系是10x倍，则写入倍数差也是10倍。

一般写放大可达10-100倍。
 