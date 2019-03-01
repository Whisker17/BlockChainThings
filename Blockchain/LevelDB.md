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

