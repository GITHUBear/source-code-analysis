# 总体结构

## 相关文件

- include/rocksdb/persistent_cache.h： 定义 PersistentCache 接口
- utilities/persistent_cache/persistent_cache_tier.h: 定义 PersistentCacheTier 描述单层 PeristentCache; PersistentTieredCache 将多级的 PersistentCacheTier 抽象为一个 PersistentCache 来使用
- utilities/persistent_cache/persistent_cache_tier.cc: 给出了 PersistentCacheTier 和 PersistentTieredCache 的默认方法实现
- utilities/persistent_cache/block_cache_tier.h: PersistentCacheTier 的一个具体实现，在 SSD 等块设备上建立 Cache
- utilities/persistent_cache/block_cache_tier_file.h: 定义了 block_cache 的 IO 抽象，以及流水线 IO 方式的实现
- utilities/persistent_cache/volatile_tier_impl.h: PersistentCacheTier 的一个具体实现，在内存中建立 Cache

## 重要结构

### PeristentCacheConfig

PersistentCacheConfig 指定了一些关于 PersistentTieredCache 的配置选项：

- env： rocksdb 封装的平台无关的系统函数
- path： block cache 所在的存储设备路径，实际是在指定路径的cache文件夹下

还有很多与 Block Cache 的 IO 流程控制有关的参数，在了解了具体流程后再做分析

### BlockCacheTier

``` C++
const PersistentCacheConfig opt_;             // BlockCache options
BoundedQueue<InsertOp> insert_ops_;           // Ops waiting for insert
rocksdb::port::Thread insert_th_;                       // Insert thread
WriteableCacheFile* cache_file_ = nullptr;    // Current cache file reference
CacheWriteBufferAllocator buffer_allocator_;  // Buffer provider
ThreadedWriter writer_;                       // Writer threads
BlockCacheTierMetadata metadata_;             // Cache meta data manager
```

PersistentCacheConfig 被专门用来指定 BlockCacheTier（有同步机制） 的配置;
insert_ops_ 是一个 InsertOp 的同步队列，有点类似于 Go 中的 channel，但是超出 bound 的时候不发生阻塞而是直接丢弃了 push 的元素; insert_th_ 执行 insertMain() 函数处理队列中的 insert 操作; cache_file_ 是对 BlockCache 写 IO 流程的抽象; buffer_allocator_ 提供 Cache 文件写入的缓冲区; writer_ 线程完成写入流程; metadata_ 记录相关的元数据

下面详细分析这几个成员：

#### WriteableCacheFile* cache_file_

为了说明 cache_file_ 的作用将按照 BlockCacheFile <- RandomAccessCacheFile <- WriteableCacheFile 的继承关系进行逐步分析：

#### BlockCacheFile

``` C++
class BlockCacheFile : public LRUElement<BlockCacheFile>
```

BlockCacheFile（有同步机制） 定义了一个接口，将自己封装为一个 LRUElement，简单来讲就是添加了 prev 和 next 指针以及引用计数，每一个 BlockCacheFile 都描述了 cache/ 文件夹下由 cache_id 为文件名， .rc 为后缀的一个数据文件，同时维护一个 *BlockInfo 的 list，记录 key 到 LBA {cache-id, offset, size} 的映射

下面看看实现了 BlockCacheFile 的 RandomAccessCacheFile：

#### RandomAccessCacheFile

RandomAccessCacheFile 对 BlockCacheFile 的 Read 虚函数进行了 Override，内部的 RandomAccessFileReader 是对 Env 的 RandomAccessFile 进行的封装，接近底层这里不做深究，只需要知道这个类提供了通过 offset 随机读取一个文件的能力（虽然提供一些统计以及缓冲/Direct IO方式等，但是对于分析 PersistentCache 来说无需深究）

然后对于进行了 Override 的 Read 方法，其核心就是通过 RandomAccessFileReader 的 Read 方法传入 LBA.offset 和 LBA.size 来读取 CacheFile 当中的 CacheRecord

然后在 RandomAccessCacheFile 上最终实现了 WritableCacheFile：

#### WriteableCacheFile

WriteableCacheFile 在 RandomAcessCacheFile 的基础上进一步提供写入缓冲池 bufs_， 以及在单个文件（或者说cache_id_）数据量达到限制的时候自动创建新的 cache_id_ 标记的文件

1. 缓冲池管理

缓存池借助 CacheWriteBufferAllocator 来实现管理，用户通过它来获得分配后的 CacheWriteBuffer，然后就可以通过 Append 方法来操作缓冲区。

2. 内存缓冲区布局以及 Append

``` plain
   +--------+       +--------+        +--------+
   |  Cache |       |  Cache |        |  Cache |
   |  Write |       |  Write |        |  Write |
   | Buffer |       | Buffer |        | Buffer |
   +--------+       +--------+        +--------+
        A                A                 A
        |                |                 |
        +---------+      |     +-----------+
                  |      |     |
               +-----+-----+-----+
               | buf1| buf2| buf3|
               +-----+-----+-----+
                        A      A
                        |      |
                        |      +-------- buf_woff: 目前已经使用的缓冲区索引
                        +--------------- buf_doff: 目前已经刷写到块设备的缓冲区索引
```

Append 是除了 Read 外继承自 BlockCacheFile 的另一个接口：

i. 在缓冲池空间不足以写入本次记录的时候扩展缓冲池 ExpandBuffer

ii. 设置 LBA 三个字段 {cache_id, offset, size}

iii. 将 cache 记录格式化写入到缓冲区， CacheRecord::Append 能够处理记录跨缓冲区的情况

iv. DispatchBuffer 将 buf_doff 索引的缓冲区交给写入线程进行刷写操作，通过 Env 的 WritableFile，使用写线程 writer_ 去执行写入（非阻塞，仅仅只是加入到等待队列中）

v. 在 writer_线程完成这个 IO 之后，执行回调函数 BufferWriteDone，可能会继续提交 buf_doff 索引的缓冲区继续刷写操作，也可能完成文件后会归还并清空 bufs_，降低引用计数，最后打开以 RandomAccess 方式进入读模式，就是上面的 RandomAccessCacheFile

3. 写入线程池 ThreadedWriter

qdepth 指定了线程池的工作线程数量， io_size 则指定了 DispatchIO 当中写入线程每一次使用 WritableFile 句柄写入文件中的数据量大小;至于 ThreadedWriter 本身是一个非常典型的简易线程池模型，这里就不做赘述了

总结一下，WriteableCacheFile 具备一个 LRUElement 的相关特性，利用了缓冲池以及线程池提供异步的 IO 流程。 回到 BlockCacheTier 来看一下它是如何在块设备上管理 CacheFile 的

如前所述，BlockCacheTier 在指定路径的 cache 文件夹下创建文件名模式为 digits.rc 的文件，下面来重点关注 BlockCacheTier 的 Insert Lookup 等操作

#### BlockCacheTier::Insert

在 Insert 中根据是否采用 pipeline_writes 模式会采用后台与非后台两种方式，启用 pipeline_writes 情况下的 IO 工作流程如下图所示：

``` plain
                                     InsertOp请求
                                          |
                                          |  入队
                                          V
                              +----+----+----+       +----+
             InsertOp 队列    |    |    |    |  …… |    | （长度由 max_write_pipeline_backlog_size 指定）
                              +----+----+----+       +----+
                                 |
                                 V
                              +--------+
                              | Insert |
                              |  线程  |
                              +--------+
                                   |
                                   +-----------+                    +------------+
                                               |                    |            |
                                               V                    |            |
                                  +-----------------------+         |   缓冲池   |
                                  |       Writeable       |   写入  |            |
               +----------------->|       CacheFile       |-------->|            |
               |                  +-----------------------+         +------------+
               |                                                           |
               |                                         +-----------------+
               |                                         |  struct IO
               |                                         V
               |                       +----+----+----+
               |               IO队列  |    |    |    |  ……  
               | 回调                  +----+----+----+
               |                          |    |
               |                          |    +---+
               |                          |        |
               |                          V        V
               |                       +-------+-------+
               |                       |       |       | ……  
               |               线程池  |       |       |
               |                       +-------+-------+
               |                            |
               +----------------------------+
```

启用 pipeline_write 与不启用的重要区别在于对于缓冲池空间被使用完的情况，启用后台线程可以等待部分 buffer 空间归还，而不启用的情况需要用户手动再次尝试

然后是 Insert 的具体流程，总体来讲比较简单：

i. 在 BlockCacheTierMetaData 的 hash 表中查找插入的 key 是否已经存在，如果存在直接结束

ii. WriteableCacheFile 进行 Append 操作，如果本次写入后，文件大小已经达到了单个文件大小的限制，就会 NewCacheFile，新的 CacheFile 的指针会加入到 MetaData 的一个 hash 表 cache_file_index 中

iii. 将 Append 返回的 LBA 与 key 加入到 block_index 中

需要注意的是 block_index 是一个无淘汰的索引结构，就是说所有的 key<->LBA 映射都会保存，而 cache_file_index 则存在着 LRU 淘汰，实现 LRU 淘汰算法的 hash 表为 EvictableHashTable，目前暂不详述

#### BlockCacheTier::Lookup

流程如下：

i. 通过 block_index 用 key 查找 LBA

ii. 上一步成功则利用 LBA 的 cache_id，在 cache_file_index 中查找

iii. 如果上一步成功则利用 BlockCacheFile 的 Read 方法读取相应记录

#### BlockCacheTier::Reserve

本方法就是在 Cache 的大小达到额定值的时候进行 Evict，会选出需要被淘汰的 BlockCacheFile 句柄，此时在 BlockCacheFile 当中用于反向索引的 BlockInfo {key，LBA} 会从 block_index 当中删除（实现中使用的是回调函数的方式实现的 RemoveAllKeys），最后文件句柄会负责删除文件

至此 BlockCacheTier 基本分析完毕

### VolatileCacheTier

VolatileCacheTier 是利用内存作为存储媒介的缓存层，不需要关注文件操作，所以总体上比 BlockCacheTier 要简单不少：

``` C++
  const bool is_compressed_ = true;    // does it store compressed data
  IndexType index_;                    // in-memory cache
  std::atomic<uint64_t> max_size_{0};  // Maximum size of the cache
  std::atomic<uint64_t> size_{0};      // Size of the cache
```

相关成员也非常少，核心就是一个 EvictableHashTable index_，所以 Insert 和 Lookup 方法主要都是在围绕 index_ 进行操作，但是 VolatileCacheTier 会检查是否存在 next_tier, 如果存在会继续向下一级 Cache 进行 Lookup 或者 evict 的数据的 insert 操作

### PersistentTieredCache及其使用

PersistentTieredCache 总体架构图如图所示：

``` plain
PersistentTieredCache architecture:
+--------------------------+ PersistentCacheTier that handles multiple tiers
| +----------------+       |
| | RAM            | PersistentCacheTier that handles RAM (VolatileCacheImpl)
| +----------------+       |
|   | next                 |
|   v                      |
| +----------------+       |
| | NVM            | PersistentCacheTier implementation that handles NVM
| +----------------+ (BlockCacheImpl)
|   | next                 |
|   V                      |
| +----------------+       |
| | LE-SSD         | PersistentCacheTier implementation that handles LE-SSD
| +----------------+ (BlockCacheImpl)
|   |                      |
|   V                      |
|  null                    |
+--------------------------+
              |
              V
             null
```

PersistentTieredCache 的应用场景应当是在 PersistentCache 在 rocksdb 中使用的场景中，比如 BlockBasedTable 中，具体使用方式在相关 test 中给出了，这里不再赘述。具体使用环境详见 table/block_based/block_based_table_reader.cc
