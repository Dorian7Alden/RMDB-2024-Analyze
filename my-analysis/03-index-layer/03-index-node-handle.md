# 03. IxNodeHandle 节点操作

`IxNodeHandle` 封装了 B+ 树单个节点的读写操作。类似记录层的 `RmPageHandle`，它把原始页面字节解析为 keys 和 rids 数组。

## 结构概览

`src/index/ix_index_handle.h:60`

```cpp
// src/index/ix_index_handle.h:60
class IxNodeHandle {
  const IxFileHdr* file_hdr;  // 所属文件的元信息
  Page* page;                 // 缓冲池中的页面
  IxPageHdr* page_hdr;        // 页内 IxPageHdr 区域指针
  char* keys;                 // 页内 keys 数组指针
  Rid* rids;                  // 页内 rids 数组指针
};
```

与记录层 `RmPageHandle` 对比：没有 bitmap，keys 和 rids 直接用数组访问，因为 B+ 树通过 `num_key` 就知道有效键的数量。

## 基本信息

```cpp
// IxNodeHandle::get_size, src/index/ix_index_handle.h:85
int get_size()             // 当前节点键数量（从 page_hdr->num_key 读取）
void set_size(int size)    // 设置键数量
int get_max_size()         // 最大容量 = btree_order + 1
int get_min_size()         // 最小容量 = get_max_size() / 2（半满要求，根节点例外）
bool isFull()              // 是否已满（num_key == get_max_size()）
```

`get_max_size()` 是 `btree_order_ + 1`，即每个节点最多容纳的键值对数。
`get_min_size()` 是 `get_max_size() / 2`，B+ 树的"半满"要求——除根节点外，节点键数必须 ≥ min_size。

## 键和 Rid 的读写

```cpp
// IxNodeHandle::get_key, src/index/ix_index_handle.h:128
char* get_key(int key_idx)   // 获取第 key_idx 个键的指针
void set_key(int key_idx, const char* key)  // 写入键（memcpy）
char* get_last_key()         // 获取最后一个键的指针

Rid* get_rid(int rid_idx)    // 获取第 rid_idx 个孩子 Rid 的指针
void set_rid(int rid_idx, const Rid& rid)   // 写入孩子 Rid
Rid* get_last_rid()          // 获取最后一个孩子 Rid 的指针
```

`key_idx` 和 `rid_idx` 的取值范围都是 `[0, num_key)`。

内部节点的 `rid` 存的是孩子节点的页面号（`Rid{child_page_no, -1}`），叶节点的 `rid` 存的是实际记录位置（指向表的记录）。

## 页面级导航

```cpp
// IxNodeHandle::get_page_no, src/index/ix_index_handle.h:98
page_id_t get_page_no()        // 当前节点所在页面号
PageId get_page_id()           // 当前节点 PageId（fd + page_no）
page_id_t get_next_leaf()      // 下一个叶节点页面号（链式遍历用）
page_id_t get_prev_leaf()      // 前一个叶节点页面号
page_id_t get_parent_page_no() // 父节点页面号
```

> **使用场景**：
> - `get_next_leaf` / `get_prev_leaf`：叶节点链表是 B+ 树范围扫描的基础（见 06 IxScan）。`split`（04b）和 `coalesce`（04c）也会维护这条链表
> - `get_parent_page_no`：分裂或合并后需要向上修改父节点时使用（`insert_into_parent`、`coalesce_or_redistribute`）

## 节点类型判断

```cpp
// IxNodeHandle::is_leaf_page, src/index/ix_index_handle.h:108
bool is_leaf_page()       // 是否为叶节点（page_hdr->is_leaf）
bool is_internal_page()   // 是否为内部节点（!is_leaf）
bool is_root_page()       // 是否为根节点（parent == IX_NO_PAGE）
```

## 页内查找：lower_bound 和 upper_bound

在节点内查找目标 key 的位置。框架中为空，需自行实现。

```cpp
// IxNodeHandle::lower_bound, src/index/ix_index_handle.cpp:48
int lower_bound(const char* target)  // 找第一个 >= target 的 key_idx
int upper_bound(const char* target)  // 找第一个 > target 的 key_idx
```

两个方法都可用二分查找实现，比较函数使用 `ix_compare`（见 [04a-btree-search.md](./04a-btree-search.md)）。

> **使用场景**：`lower_bound` 是节点内查找的基石，被以下方法调用：
> - `leaf_lookup`（04a）——叶节点内精确查找 key
> - `internal_lookup`（04a）——内部节点定位下一个孩子
> - `insert`（本页）——找到插入位置
> - `remove`（本页）——找到删除位置
> 
> 树级的 `lower_bound` / `upper_bound`（见 04a）则是对外暴露的范围扫描入口，`IxScan`（06）用它们确定扫描的起止边界。

## 插入与删除

在节点内操作键值对。框架中为空，需自行实现。

```cpp
// IxNodeHandle::insert_pairs, src/index/ix_index_handle.cpp:142
void insert_pairs(int pos, const char* key, const Rid* rid, int n)
// 在 pos 位置插入 n 个连续键值对，需 memmove 腾出空间

void insert_pair(int pos, const char* key, const Rid& rid)
// 在 pos 位置插入单个键值对（调用 insert_pairs）

std::pair<int, int> insert(const char* key, const Rid& value)
// 先查找插入位置（lower_bound），重复则跳过，否则调用 insert_pair

void erase_pair(int pos)
// 删除 pos 位置的键值对，需 memmove 填补空缺

std::pair<int, int> remove(const char* key)
// 查找并删除指定 key 的键值对，返回删除后的数量和位置
```

> **使用场景**：这些方法本身只操作当前节点，不关心树的结构变化。它们被更高层的方法调用：
> 
> | 方法 | 被谁调用 | 场景 |
> |------|---------|------|
> | `insert` | `insert_entry`（04b） | 找到叶节点后，先插入键值对 |
> | `insert_pairs` | `split`（04b） | 将节点右半部分搬到新节点 |
> | `insert_pairs` | `insert_into_parent`（04b） | 将新节点的第一个 key 插入父节点 |
> | `insert_pair` | `redistribute`（04c） | 从兄弟节点借一个键值对 |
> | `remove` | `delete_entry`（04c） | 找到叶节点后，先删除键值对 |
> | `erase_pair` | `coalesce`（04c） | 合并后删除父节点中对应的分隔键 |
> | `erase_pair` | `redistribute`（04c） | 重分配后删除兄弟的旧键 |
> 
> 换句话说：**insert/erase 是底层工具，split/coalesce/redistribute 用它们来重塑树的结构**。

## isSafe：锁缩放的判断

B+ 树可能被多个查询同时访问。如果一次插入把根节点到叶节点的整条路径都锁住不释放，其他查询全部要排队等——并发性能极差。

锁缩放（latch crabbing）解决这个问题：从根往下走的过程中，如果判断当前节点**不会发生结构变化**（不会分裂或合并），那它上面的祖先节点也不会受影响，可以提前释放祖先的锁。

`isSafe` 就是做这个判断的——"当前节点是否安全"：

```cpp
// IxNodeHandle::isSafe, src/index/ix_index_handle.cpp:26
bool isSafe(Operation operation) {
  if (!is_root_page() && operation == INSERT)
    return get_size() + 1 < get_max_size();  // 插入后仍不满 → 不会分裂
  if (!is_root_page() && operation == DELETE)
    return get_size() > get_min_size();       // 删除后不亏空 → 不会合并
  return true;  // 根节点或 FIND 操作
}
```

| 条件 | 含义 | 为什么算"安全" |
|------|------|---------------|
| `operation == INSERT` 且 `size + 1 < max` | 插入后还没满 | 不会触发分裂，父节点结构不受影响 |
| `operation == DELETE` 且 `size > min` | 删除后还够半满 | 不会触发合并或重分配 |
| `is_root_page()` | 当前是根 | 根没有父节点可释放，一律返回 true |
| `operation == FIND` | 只读操作 | 不改变结构，一定安全 |

重点区分：`size + 1 < max` 是安全（插入后仍不满），`size + 1 == max` 是**不安全**——再插入一个就会分裂，当前虽然能插入但下一次操作就可能触发结构变化。

> **使用场景**：`isSafe` 被 `find_leaf_page` 中的锁缩放流程调用。完整的锁缩放流程（加锁步骤伪代码 + 三层 B+ 树具体例子）见 [04a-btree-search.md](./04a-btree-search.md)。

## 源码对应

| 内容 | 文件 | 行号 |
|------|------|------|
| IxNodeHandle 完整定义 | `src/index/ix_index_handle.h` | 60-208 |
| 框架待实现方法 | `db2026-x/src/index/ix_index_handle.cpp` | 21-141 |

上一节：[02-index-data-structures.md](./02-index-data-structures.md) | 下一节：[04a-btree-search.md](./04a-btree-search.md)
