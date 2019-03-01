# LevelDB

**一句话说，LevelDB是一款基于LSM树的持久化存储的KV数据库，具有极佳的写功能。**

## 整体架构

![LevelDB_1](./pics/LevelDB_1.png)

如上图所示，LevelDB共有六个成分组成：

1. Memtable
2. Immutable memtable
3. Log
4. Sstable
5. Manifest
6. Current

## 基本概念

### Memtable

DB数据在内存中的存储方式，写操作会先写入memtable，memtable是处在内存中的，所以效率非常高，但是由于内存容量是有限的，并且价格比较贵，所以我们会在一定范围之后进行一个持久化的过程，相关内容我们在下文继续介绍。