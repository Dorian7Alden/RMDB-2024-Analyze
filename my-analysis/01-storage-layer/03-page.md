# 03. 页面数据结构（Page）

## 概述

Page 是 RMDB 中数据存储和交换的**最小单位**。无论磁盘文件还是内存缓存，数据都以"页"为单位组织。理解 Page 的结构是理解整个存储层的前提。

## 前置知识

- **为什么需要"页"？** 磁盘读写的最小单位是扇区（通常 512 字节），但逐扇区读写效率低。DBMS 将多个扇区合并为一个"页"（通常 4KB~16KB），作为与磁盘交换数据的基本单位。这就像搬家时不会一件一件拿，而是装箱搬运。

## 数据结构

### PageId

页面由"哪个文件 + 文件中第几页"共同定位（`page.h:21-24`）：

```cpp
struct PageId {
    int fd;                           // 文件描述符
    page_id_t page_no = INVALID_PAGE_ID;  // 页号，类型为 int32_t
};
```

**实例**：student 表的数据文件 fd=3，该文件第 2 页的 PageId = `{fd: 3, page_no: 2}`。

> **关键理解**：`page_no` 不是全局唯一的，必须配合 `fd` 才有意义。两个不同文件的第 0 页是两个完全不同的页面。

### Page

```cpp
class Page {
    PageId id_;                       // 页面唯一标识
    char data_[PAGE_SIZE] = {};       // 4KB 的实际数据
    bool is_dirty_ = false;           // 是否为脏页（内存中被修改过，需写回磁盘）
    int pin_count_ = 0;               // 被引用的次数（>0 时不能被淘汰）
    RWLatch rwlatch_;                 // 读写锁（参考实现新增）
};
```

**页面内存布局**（4KB = 4096 字节）：

```
data_[0]   data_[1]   data_[2]   data_[3]   data_[4]  ...  data_[4095]
┌──────────┬──────────────────────────────────────────────────────────┐
│  LSN     │  页面头 (Page Hdr)    │        实际记录数据               │
│ (4字节)  │  (从 OFFSET_PAGE_HDR  │       (剩余空间)                 │
│          │   = 4 开始)           │                                  │
└──────────┴──────────────────────────────────────────────────────────┘
```

三个关键偏移常量（`page.h:85-87`）：

| 常量 | 值 | 含义 |
|------|-----|------|
| `OFFSET_PAGE_START` | 0 | 页面数据的起始位置 |
| `OFFSET_LSN` | 0 | LSN（Log Sequence Number，日志序列号）的偏移，占前 4 字节 |
| `OFFSET_PAGE_HDR` | 4 | 页面头的起始偏移（LSN 之后） |

> **LSN 是什么？** Log Sequence Number（日志序列号），用于崩溃恢复时判断页面是否已更新。这一般属于恢复层的概念，存储层只需要知道它占页面前 4 字节。

## 框架 vs 参考实现差异

| 方面 | 框架 `db2026-x/` | 参考实现 `src/` |
|------|-----------------|-----------------|
| `operator!=` | 未定义 | 已定义，方便比较 PageId |
| `PageId::Get()` | 有 `(fd << 16) \| page_no` | 注释掉了，不需要 |
| `PageIdHash` | 有自定义结构体 | 改用 `std::hash<PageId>` 特化 |
| `RWLatch` | 无 | `RWLatch rwlatch_` 成员 + `WLatch/WUnlatch/RLatch/RUnlatch` 方法 |
| `friend class` | `BufferPoolManager` | `BufferPoolInstance` |
| `get_pin_count()` | 无 | 有，返回 `pin_count_` |

**最关键的变化**：框架中 Page 的 `friend class` 是 `BufferPoolManager`，参考实现中改为了 `BufferPoolInstance`。这暗示了核心架构变化——从一个大 BufferPool 管理所有 Page，变为多个 Instance 各管一部分。

## Page 的生命周期

一个 Page 对象的状态由 `pin_count_` 控制：

```
pin_count_ = 0  →  空闲状态，可以被淘汰（淘汰时如果是脏页需先写回磁盘）
pin_count_ > 0  →  被引用中，不能被淘汰
```

**实例**：一次全表扫描中的 Page 生命周期：

```
1. Executor 请求 student 表第 1 页
2. Buffer Pool 调用 fetch_page({fd:3, page_no:1})
   → 找到空闲 frame，从磁盘读入 → pin_count_ = 1
3. SeqScanExecutor 遍历该页中的记录
4. 该页记录遍历完毕 → unpin_page → pin_count_ = 0
5. 后续 LRU Replacer 可以选择淘汰此页
```

## 小结

- `PageId = fd + page_no`，两者结合才能唯一标识一个页面
- Page 的 `data_[PAGE_SIZE]`（4096 字节）是实际存储数据的区域，前 4 字节是 LSN
- `pin_count_` 控制页面的生命周期：>0 受保护，=0 可淘汰
- 框架版 Page 缺少 RWLatch 和 `get_pin_count()`，参考实现补全了并发控制能力

下一节：[04. 缓冲池概述](./04-buffer-pool-overview.md)
