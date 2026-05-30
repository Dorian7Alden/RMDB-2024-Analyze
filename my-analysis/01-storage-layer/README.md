# 第 1 章：存储层

存储层是 DBMS 的基石——数据最终都要落到磁盘上，而存储层负责管理"磁盘上怎么存、内存里怎么缓存、两者之间怎么搬运"。

## 框架 vs 参考实现

| 模块 | 框架 `db2026-x/` | 参考实现 `src/` | 学习目标 |
|------|-----------------|-----------------|----------|
| Disk Manager | 已完整实现 | 同框架（仅格式差异） | 简要了解 |
| Page | 已给基础结构 | 增加了 RWLatch 集成 | 掌握数据结构 |
| Buffer Pool Manager | 简单的单实例骨架 | 多实例分区 + LRU | **核心学习目标** |
| Buffer Pool Instance | 不存在 | 完整实现 | **需要从零实现** |
| Page Guard | 不存在 | 完整实现 | **需要从零实现** |
| RWLatch | 不存在 | 完整实现 | **需要从零实现** |

## 本章目录

| 序号 | 文档 | 内容 | 类型 |
|------|------|------|------|
| 01 | [磁盘管理器](./01-disk-manager.md) | 磁盘文件的读写、分配、管理 | 了解 |
| 02 | [文件头](./02-file-header.md) | RmFileHdr、IxFileHdr、磁盘布局、读写路径 | 了解 |
| 03 | [页面数据结构](./03-page.md) | PageId、Page 的结构与内存布局 | 掌握 |
| 04 | [缓冲池概述](./04-buffer-pool-overview.md) | 为什么要缓冲池、核心概念（pin/unpin/脏页） | 理解 |
| 05 | [单实例缓冲池](./05-buffer-pool-single.md) | 框架给的基础版本：page_table、free_list、LRU | 掌握 |
| 06 | [多实例缓冲池](./06-buffer-pool-multi.md) | 分区并发优化：BufferPoolInstance | **核心** |
| 07 | [LRU 页面替换](./07-buffer-pool-lru.md) | LRU Replacer 算法实现 | **核心** |
| 08 | [Page Guard](./08-page-guard.md) | RAII 页面守卫：自动 unpin | **核心** |
| 09 | [RWLatch 读写锁](./09-rwlatch.md) | 页面级读写锁 | **核心** |

## 存储层在整体架构中的位置

```
查询处理层 (Executor)
  │
  ▼
数据管理层 (RM / IX)
  │
  ▼
存储层     ← 本章 ← Buffer Pool + Disk Manager + Replacer + Page
  │
  ▼
磁盘文件 (*.db)
```
