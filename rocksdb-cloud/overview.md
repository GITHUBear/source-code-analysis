# DBCloudImpl

继承线路为 DBCloudImpl <- DBCloud <- StackableDB <- DB，StackableDB 是 rocksdb 提供的对数据库内核的封装以便于在内核的基础上添加新功能，DBCloud 定义了 RocksDB-Cloud 的特有接口，包括云环境下的打开数据库方法，Remote Compaction 以及数据迁移方法等

## Remote Compaction

### 目的

RocksDB 中 Compaction 是一个计算代价较高的操作，一个稳定大小的数据库如果需要提高写入速度，就需要相应地提高 Compaction 的计算速度，在单个节点上是很难实现这两者的匹配的，所以在 RocksDB-Cloud 当中提出了 Remote Compaction 将计算和存储分离，提出了如下所示的架构：

``` plain
        +----------+       1. Compaction Request     +-----------+
        |  RocksDB |-------------------------------->| Compactor |
        |   local  |<--------------------------------|   Server  |
        +----------+       4. Compaction Result      +-----------+
            A                                             A
            |                                             | 2,3. fetch & Write
            |                                             |
            V                                             V
        +--------------------------------------------------------+
        |                         S3                             |
        +--------------------------------------------------------+
```

不过，感觉这个功能并不是针对 Cloud 环境必须做出的实现。在去掉 Compactor Server 的情况下，仅仅通过 RocksDB local 与 S3 进行交互在 LSM-tree 的一致性维护上也并不存在问题，主要是仅使用 local 进行 Compaction 的时候在云环境中会访问较缓慢的 S3 从而导致 Compaction 极有可能跟不上写入负载的速度，从而导致 Write Stall 甚至 Write Stop 的发生

### 核心方法

Remote Compaction 的核心方法定义在 db_impl_remote_compaction.cc 当中，分别是 doCompact 以及 ExecuteRemoteCompactionRequest 方法，在 ExecuteRemoteCompactionRequest 内部调用 doCompact 方法来构造 CompactionJob 并最终完成压缩操作。

### doCompact

本方法的实现和 CompactFiles 这个 manual compaction 的实现非常的相似，通过指定待压缩的SST文件的文件名以及 output_level，在指定的 cfd 的指定 version 上进行压缩操作。实现中采用的方式同样是通过 CompactionPicker 创建 CompactionJob 对象，调用 Prepare 最终调用 run 方法完成 Compaction 操作

### ExecuteRemoteCompactionRequest

ExecuteRemoteCompactionRequest 本方法就是在 doCompact 基础上处理了一下 doCompact 需要的参数，并在 doCompact 操作失败的时候删除掉失败 Compaction 操作过程中产生的无效文件，对于 doCompact 处理完成后在 Compactor Server 上产生的文件的元数据信息通过 PluggableCompactionResult 进行返回，同时为了能够方便 RocksDB local Server 能够在云存储中访问新的文件，在 db_cloud_impl.cc 的同名方法下，对 PluggableCompactionResult 的输出文件路径进行了转换（`在转换中使用到了 rocksDB-cloud 实现的 cloud env，其中使用了一个名为 epoch 的概念，由于对于云存储尚不熟悉，还没有深究`）

了解到这里的时候，突然有个疑问：RocksDB Local 在使用了 Remote Compaction 的情况下，是否还会允许 auto compaction 的触发？但是由于 DBCloudImpl 继承了 DBImpl，可以发现 DBCloudImpl 并没有自己实现 MaybeScheduleFlushOrCompaction，基本可以断定 DBCloudImpl 也会使用 auto compaction 来完成压缩任务的，只不过底层的 env 使用的是 CloudEnv，文件的读写都是通过云存储环境（`是否意味着 SST 文件数据是全部保存在云存储上的？`）

所以 Remote Compaction 其实应该是一种比较特殊的 manual compaction 方式，因为在整个项目内并没有 ExecuteRemoteCompactionRequest 继续被上层方法调用的证据，所以应该是一种用户在使用本嵌入式数据库时的一种接口，Remote Compaction 带来的好处是：

- 当 RocksDB local 的 auto compaction 因为访问云环境不能与当前的高写入负载匹配的时候，可以通过 Remote Compaction 通知一个 Compactor Server 以 Ghost 模式打开一个 RocksDB，然后在 Compactor Server 上读取 S3 数据文件并完成压缩任务

- 上述的处理过程是比较弹性且动态的，能够快速地对当前的负载进行反应，无需多个 Server 之间事先传递 SST 数据的拷贝

可以发现 Remote Compaction 的实现很大程度上依赖于 zero copy clone 的性质，下面简单看看 DBCloud 是如何进行 CloneDB 的

### 一些可能存在问题的结论

- RocksDB Local Server 和 Compactor Server 是不需要配置在云上的（如果这个结论是对的，xp学长的设计目标是否需要做到 Local 节点上云？）

- Remote Compaction 方式并没有取代默认的 auto compaction 方式，只不过 auto compaction 的底层文件读写更换为了 CloudEnv，Remote Compaction 是类似于 CompactFiles 的外部使用的接口，来满足较高写入负载时的压缩速度（或者说不需要 Remote Compaction 也可以运行）

- 在 RocksDB-Cloud 中 SSTable 文件应该都保存在了云存储中，本地通过 Persistent Cache 在本地缓存从云存储中访问的数据

## CloneDB

在 db_cloud_test.cc 当中为了便于测试实现了一个 CloneDB 方法

## DBCloudImpl::Open

``` C++
Status DBCloud::Open(const Options& options, const std::string& dbname,
                     const std::string& persistent_cache_path,
                     const uint64_t persistent_cache_size_gb, DBCloud** dbptr,
                     bool read_only);

Status DBCloud::Open(const Options& opt, const std::string& local_dbname,
                     const std::vector<ColumnFamilyDescriptor>& column_families,
                     const std::string& persistent_cache_path,
                     const uint64_t persistent_cache_size_gb,
                     std::vector<ColumnFamilyHandle*>* handles, DBCloud** dbptr,
                     bool read_only)
```

Open 方法有两个重载版本以是否配置列族区分，对比 RocksDB 默认的 DBImpl 的参数集，可以发现参数中多出了 persistent_cache_path, persistent_cache_size_gb 以及 read_only, 在 DBImpl 中 persistent_cache 是可选的配置项，可见在 DBCloud 中是强制使用的功能

和 DBImpl 类似的，无列族描述器指定的 Open 方法，内部也是构造 cf_options, 然后创建一个默认列族的描述器，交给配置列族的方法进行处理