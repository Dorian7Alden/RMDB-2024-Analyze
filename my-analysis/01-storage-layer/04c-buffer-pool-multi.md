# 04c. 多实例缓冲池（参考实现）

## 问题回顾

单实例版本用一把 `std::mutex latch_` 锁住所有操作——16 个线程访问 16 个不同的页面也必须排队。参考实现的核心优化思路：**把一个大池子拆成多个小池子，每个小池子有自己独立的锁**。

## 数据结构

### BufferPoolManager

```cpp
// src/storage/buffer_pool_manager.h
class BufferPoolManager {
  size_t pool_size_;                                       // 总容量 = 65536
  BufferPoolInstance* instances_[BUFFER_POOL_INSTANCES]{}; // 16 个分区
  std::hash<PageId> hasher_;                               // 哈希函数：决定页面分到哪个 instance
  DiskManager* disk_manager_;
  LogManager* log_manager_;
};
```

关键的 `get_instance_no` 方法决定页面归属：

```cpp
// buffer_pool_manager.h:106-108
inline std::size_t get_instance_no(const PageId& page_id) {
    return hasher_(page_id) % BUFFER_POOL_INSTANCES;  // 哈希取模，分到 16 个分区之一
}
```

**实例**：student 表有 25 页，它们可能分布在不同 instance 中：

```
{fd:3, page_no:0}  → hash → instance 3
{fd:3, page_no:1}  → hash → instance 7
{fd:3, page_no:2}  → hash → instance 3
{fd:3, page_no:3}  → hash → instance 0
...
```

**核心效果**：不同 instance 的操作可以并行！线程 A 访问 instance 3 的页面，线程 B 访问 instance 7 的页面——两把不同的锁，互不阻塞。

### fetch_page

参考实现的 `BufferPoolManager::fetch_page` 非常简洁：

```cpp
// buffer_pool_manager.cpp:12
Page* BufferPoolManager::fetch_page(PageId page_id) {
    return instances_[get_instance_no(page_id)]->fetch_page(page_id);
}
```

就是一句话：算出分区编号，把请求**路由**过去。

### BufferPoolInstance

每个 Instance 的结构和框架的单实例版本**完全一样**：

```cpp
// src/storage/buffer_pool_instance.h
class BufferPoolInstance {
  size_t pool_size_;                                      // 本分区容量 = 65536/16 = 4096
  Page* pages_;                                           // Page 数组（大小 4096）
  std::unordered_map<PageId, frame_id_t> page_table_;     // PageId → frame_id
  std::list<frame_id_t> free_list_;                       // 空闲 frame 链表
  DiskManager* disk_manager_;
  ClockReplacer* replacer_;                               // 时钟替换器（注意：不是 LRU！）
  std::mutex latch_;                                      // 本分区的锁
};
```

> **为什么多实例版本改用 ClockReplacer？**
>
> `ClockReplacer`（时钟替换器）和 `LRUReplacer` 的目的完全一样——缓冲池满时选一个"受害者"frame 淘汰。区别在于实现方式：
>
> | | LRUReplacer（框架） | ClockReplacer（参考实现） |
> |---|---|---|
> | 数据结构 | 双向链表 + 哈希表 | 两个固定数组 + 一个游标指针 |
> | 选 victim | 取链表尾部（最久未用），O(1) | 游标循环扫描，给"二次机会"，近似 LRU |
> | 内存分配 | 动态（链表节点随 unpin 增减） | 静态（数组大小 = 分区帧数，编译期确定） |
> | 内部锁 | 自带 `std::mutex` | 无锁（Instance 的 `latch_` 已保护） |
>
> 核心思路：LRU 用精确排序（链表头部=最近用过，尾部=最久未用），但需要动态分配链表节点；Clock 用近似策略——一个游标循环扫描所有 frame，遇到 `pin_ == true` 的给它"第二次机会"（清零后跳过），遇到 `pin_ == false` 的就淘汰。Clock 虽然不那么精确，但**零动态分配、零内部锁开销**，更契合多实例版本"每个小池子独立加锁、分区固定"的架构。
>
> 详细实现留到 [05a. LRU 页面替换](./05a-replacer-lru.md) 展开。
>
> 每个 Instance 内部的 `fetch_page`、`unpin_page`、`find_victim_page`、`update_page` 逻辑与单实例版本几乎一致——只是 scope 从"全局"变成了"本分区"。

## 架构变化总结

```
框架单实例:                        参考实现多实例:
                                    BufferPoolManager (入口，路由)
BufferPoolManager                       │
  │  pages_[0..65535]          ┌────────┼────────┬──────────┐
  │  page_table_               │        │        │          │
  │  free_list_          Instance_0  Instance_1  ...  Instance_15
  │  latch_ (全局锁!)      │ 4096页    │ 4096页         │ 4096页
  │                       │ 自己的锁   │ 自己的锁        │ 自己的锁
  └─ 所有请求串行           └─ 并行！────┴───────────────┘
```

## 多实例并发的实际效果

```
线程 A: fetch_page({fd:3, page_no:0})  → hash → instance 3  → 获取 latch_3
线程 B: fetch_page({fd:4, page_no:0})  → hash → instance 7  → 获取 latch_7  (并行！)
线程 C: fetch_page({fd:3, page_no:1})  → hash → instance 3  → 等待 latch_3  (排队，同分区)
线程 D: fetch_page({fd:3, page_no:5})  → hash → instance 0  → 获取 latch_0  (并行！)
```

## 框架 vs 参考实现对照表

| 方面 | 框架 `db2026-x/` | 参考实现 `src/` |
|------|-----------------|-----------------|
| 池结构 | 1 个大池（65536 页） | 16 个小池（各 4096 页） |
| 锁粒度 | 1 把全局锁 | 16 把分区锁 |
| 并发度 | 串行 | 最多 16 路并行 |
| 替换策略 | `LRUReplacer` | `ClockReplacer` |
| Page 归属 | `friend BufferPoolManager` | `friend BufferPoolInstance` |
| 入口方法 | 直接实现 | 路由到对应 instance |

## 小结

- 多实例版本的核心思想：**分而治之**。把一个大池子按 PageId 哈希分成 16 个小池子
- 每个 Instance 内部的逻辑（fetch/unpin/victim/update）与单实例版本相同
- `BufferPoolManager` 退化为一个"路由器"，只负责 `hash(page_id) % 16`
- 不同 Instance 之间无竞争，并发能力提升 16 倍（理想情况）

下一节：[05a. LRU 页面替换](./05a-replacer-lru.md)
