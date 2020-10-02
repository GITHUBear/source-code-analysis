# Utils

## Bitmap

在 `Octopus` 当中实现的 bitmap 设计上比较简短，这里不做详述

其作用是在底层的文件系统布局当中记录 inode 以及 data block 的分配情况

## Lock

为了提供分布式事务，实现了 `LockService` 来进行加锁操作。
主要通过原子操作来完成，使用 CAS 技术向指定的 LockAddress 位置写入一个特殊的 key，
该 key 由 node_id 和 write_id 组成

- node_id：应该是指存储节点的编号
- write_id：写线程 id

## Table

使用常用的空闲链组织形式实现了一个内存 buffer pool

## Global




