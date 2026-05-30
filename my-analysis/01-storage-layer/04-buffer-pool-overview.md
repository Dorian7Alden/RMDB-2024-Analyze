# 04. 缓冲池概述

## 问题：为什么不直接从磁盘读？

假设没有缓冲池，每次需要数据都直接读磁盘：

```
Executor 遍历 student 表 1000 条记录
  → 每条记录都要触发一次 read_page()
    → 每次 read_page() 都要 lseek + read 系统调用
      → 1000 次磁盘 I/O
```

磁盘 I/O 的速度大约是内存访问的 **10万倍** 慢。所以需要一个**缓冲池（Buffer Pool）**——在内存中缓存磁盘页，大部分请求命中内存，少数未命中才走磁盘。

```
无缓冲池:  Executor → Disk (每次都要磁盘 I/O)
有缓冲池:  Executor → Buffer Pool → [大概率命中内存]
                                    [小概率未命中 → Disk]
```

## 核心概念

### 帧（Frame）与页（Page）的关系

```
缓冲池（内存中的大数组）
┌─────────┬─────────┬─────────┬─────────┬─────────┐
│ Frame 0 │ Frame 1 │ Frame 2 │ Frame 3 │  ...    │  ← 共 BUFFER_POOL_SIZE 个 frame
│  Page   │  Page   │  Page   │  Page   │         │
└─────────┴─────────┴─────────┴─────────┴─────────┘
     ↑                    ↑
  存了 student 表       存了 course 表
  第 0 页                第 3 页
```

- **帧（Frame）**：缓冲池数组的一个槽位，对应内存中一个 Page 对象
- **页（Page）**：逻辑概念，可能当前在某个 Frame 中，也可能只在磁盘上
- `BUFFER_POOL_SIZE = 65536`（`common/config.h:39`），即最多缓存 65536 个页面（约 256MB）

### Pin 与 Unpin

"Pin"（钉住）是缓冲池最核心的机制：

- **Pin（`pin_count_++`）**：我正在用这个页面，**不要淘汰它**
- **Unpin（`pin_count_--`）**：我用完了，**可以淘汰了**
- **pin_count 可以大于 1**：多个执行器可能同时在读同一页

```
进程 A: fetch_page → pin_count_ = 1
进程 B: fetch_page（同一页）→ pin_count_ = 2  ← 两个进程共用
进程 A: unpin_page → pin_count_ = 1
进程 B: unpin_page → pin_count_ = 0            ← 可以淘汰了
```

### 脏页（Dirty Page）

如果页面的数据在内存中被修改了，它就和磁盘上的版本不一致了。这种页面叫**脏页**。

- `is_dirty_ = true`：内存中改了，磁盘上还是旧的，**淘汰前必须先写回磁盘**
- `is_dirty_ = false`：内存和磁盘一致，直接丢弃即可

```
1. fetch_page({fd:3, page_no:1})  → 从磁盘读入，is_dirty_ = false
2. 修改该页某条记录               → is_dirty_ = true（脏了！）
3. unpin_page                     → 如果此时要淘汰，必须 write_page 写回磁盘
```

### 命中与未命中

```
fetch_page(page_id)
  │
  ├─ page_table_ 中有记录？ → 命中（Hit）→ 直接返回 Page*，pin_count++
  │
  └─ page_table_ 中无记录？ → 未命中（Miss）
       │
       ├─ free_list_ 中有空闲 frame？ → 拿来用
       │
       └─ free_list_ 空？ → 调用 Replacer 选一个受害者（Victim）
            │
            ├─ 受害者是脏页？ → 先 write_page 写回磁盘
            │
            └─ 从磁盘 read_page 读入目标页，更新 page_table_
```

## 缓冲池的核心数据结构

不管单实例还是多实例版本，缓冲池都围绕以下结构：

| 结构 | 类型 | 作用 |
|------|------|------|
| `pages_` | `Page[]` 数组 | 存放所有 Page 对象（Frame 数组） |
| `page_table_` | `unordered_map<PageId, frame_id_t>` | PageId → Frame 编号的映射，用于快速查找 |
| `free_list_` | `list<frame_id_t>` | 空闲 Frame 编号链表 |
| `replacer_` | `Replacer*` | 页面替换策略（LRU），决定淘汰哪个非空闲页 |

## 实例

以 student 表为例，页面大小为 4KB，每条记录约 100 字节，一页大约放 40 条记录。

```
student 表有 1000 条记录，共 1000/40 = 25 页（page_no 0~24）

缓冲池初始状态: free_list_ = [0, 1, 2, ..., 65535]

全表扫描开始:
  fetch_page({fd:3, page_no:0}) → 命中 free_list_[0] → frame 0 装入第 0 页
  fetch_page({fd:3, page_no:1}) → 命中 free_list_[1] → frame 1 装入第 1 页
  ...
  每读完一页: unpin_page → pin_count 归零 → Replacer 可以回收

当缓冲池满了（free_list_ 为空），再 fetch_page 时:
  Replacer 选出最久未使用的 frame → 如果是脏页先写回 → 装入新页
```

## 小结

| 概念 | 一句话 |
|------|--------|
| 缓冲池 | 内存中缓存磁盘页的大数组，减少磁盘 I/O |
| Pin/Unpin | 引用计数机制，pin_count > 0 时页面受保护 |
| 脏页 | 内存数据与磁盘不一致，淘汰前必须写回 |
| 命中/未命中 | 页面已在/不在缓冲池中 |
| Replacer | 缓冲池满时决定淘汰哪个页面 |

下一节：[05. 单实例缓冲池](./05-buffer-pool-single.md) 将讲解框架给出的基础版本实现。
