# Octopus

## 关键字

- NVM
- RDMA
- 分布式文件系统
- 无中心元数据服务器

## 概要

Octopus 结合 NVM 与 RDMA 两种技术优势设计了一个分布式文件系统

1. 提到分布式系统首先应该想到的是在引入新的技术和硬件环境下应该如何设计中间件，
在 Octopus 中设计了一种 self-identified RPC
2. 其次就是设计新的分布式事务机制来适应新的硬件，最大程度实现软硬件的契合

## 创新点

- 目前使用 RDMA 来提升网络通信效率的 DFS 主要的实现方式是使用 RDMA 
库来代替原有的网络通信模块，主要的问题就是 fs 层和网络层的跨多层软件逻辑在 
RDMA 带来的高速通信情况下成为了性能瓶颈。 Octopus 带来了更好的解决方案。
- 摆脱传统的在 local FS 上建立 DFS 架构带来的延迟较高，带宽较低的问题
- 相对应地设计了新的 RPC 机制

## 出发点

- 数据在内存中多次拷贝，诸如用户缓冲区、fs 缓冲区、网络通信在内核中使用的缓冲区等
- 高性能网络带来单位时间较多的 Request，将会给服务器 CPU 带来更大的压力
- 传统 RPC 延迟较高
- 分布式系统为了维护一致性在分布式事务上的开销不可忽视

## 系统设计

### Server Data Layout

``` plain
                                             
    +------------+--------------+--------------+--------------+--------+-------+
    |            |    Message   |    MetaData  |    MetaData  |   Data |  Log  |
    | SuperBlock |              |     Index    |              |        |       |
    |            |     Pool     |     Zone     |      Zone    |   Zone |  Zone |
    +------------+--------------+--------------+--------------+--------+-------+
                         ^               ^             ^           ^        ^
                         |               |             |           |        |
                         |               |             |           |        |
                    tmp storage     index inode      inodes       data    txn log
                    for RPC         addr with                     block   for consistency
                                    hash
```

其中，metadata 走 RPC 进行访问，而 data block 通过 RDMA 技术进行直接访问

### Shared Persistent Memory Pool

减小数据副本的拷贝

- 移除层层堆叠的 fs 结构
- 无需 page-cache 和用户 buf 通过 RDMA 技术进行数据访问
- Octpus 直接管理 raw data format 无需文件系统的帮助

### Client-Active Data IO

将服务器的数据传输压力通过 RDMA 技术转移到 client 上

#### 流程

i. client 以 RPC 的形式发送读写请求到 server 端

ii.  返回文件 inode 地址等元数据信息

iii. client 基于返回的元数据信息，发起 RDMA 读写请求

``` plain
ps: 文件访问的并发安全由 Server 和 client 一同来支持，
    Server 负责加锁，通过 GCC 读写锁来实现每个文件的互斥访问
    Client 负责解锁，通过 RDMA 原子原语实现解锁
```

#### TODO

Server 加锁而 Client 解锁的操作是否安全？

Client 的崩溃是否会导致 Server 端的死锁？

待阅读代码 [Octopus](https://github.com/thustorage/octopus)


### Self-identified MetaData RPC

#### Message-based RPC VS Memory-based RPC

Message-based RPC

优势：类似 socket 一次通信需要 Server 与 Client 之间的合作

缺点：相对较高的时延与较低的吞吐率

Memory-based RPC

优势：低时延

缺点：服务器端需要对消息缓冲区进行轮询，带来 CPU 的高开销

#### Write_with_imm

- 携带一个立即数，在 Octopus 中立即数由 Client 的 node_id 与接收缓冲区的 offset 组成
- 会在接收端消耗一个 recv request 从而可以得到 send/recv 方式的快速响应

#### TODO

可能需要了解并且实际使用一下 RDMA 库

### Collection-Dispatch Txn

传统的分布式事务技术是 2PC，需要依赖分布式日志以及日志持久化等

Octopus 的优化方案是使用 RDMA 原子操作来实现一个新的分布式事务机制

#### 崩溃一致性

Octopus 选择 `不让 participator 持有日志副本`，
仅在 Coordinator 保存 local log

在 Collection 阶段发送 Collect RPC，从 participator 收集 WriteSet

在 dispatch 阶段通过 RDMA write 对 participator 的数据进行更新，然后使用 RDMA 原语解除 
participator 的 lock

#### 并发控制

与 `Client-Active Data IO` 部分所述类似

#### TODO

- 同样是 Coordinator 端来解除 participator 的 lock 是否合适的问题
- 如果上面的问题确实存在，是否是本 DFS 牺牲了可用性 A，而来保证一致性、分区容忍？
- RDMA 的写入的数据在持久化到 NVM 的时候是否保证崩溃原子性？

### 后续优化点

- Hash-based 文件分布，对 NVM 损耗均衡并不友好
- 同一个文件的元数据以及数据块均分布在同一个服务节点上

