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

- 移除层层堆叠的 fs 结构
- 无需 page-cache 和用户 buf 通过 RDMA 技术进行数据访问
- Octpus 直接管理 raw data format 无需文件系统的帮助

### Client-Active Data IO

### Self-identified MetaData RPC

### Collection-Dispatch Txn

### 后续优化点

- Hash-based 文件分布，对 NVM 损耗均衡并不友好
- 同一个文件的元数据以及数据块均分布在同一个服务节点上

