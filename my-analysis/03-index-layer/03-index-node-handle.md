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

两个方法都可用二分查找实现，比较函数使用 `ix_compare`（见 [01-index-layer-overview.md](./01-index-layer-overview.md)）。

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

## isSafe：锁缩放的判断

### 先理解：为什么要锁缩放

B+ 树可能被多个查询同时访问。如果一次插入把根节点到叶节点的整条路径都锁住不释放，其他查询全部要排队等——并发性能极差。

锁缩放（latch crabbing）解决这个问题。名字很形象：像螃蟹走路，**抓住下一级节点后，才松开上一级**。

核心思路：从根往下走的过程中，如果判断当前节点**不会发生结构变化**（不会分裂或合并），那它上面的祖先节点也不会受影响，可以提前释放祖先的锁。

代码在 `find_leaf_page`，`src/index/ix_index_handle.cpp:270`。

### isSafe：判断"当前节点是否安全"

"安全"的意思是：操作不会导致当前节点分裂或合并。

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
| `operation == INSERT` 且 `size + 1 == max` | 插入后刚好满 | **不安全**——再插入一个就会分裂，但这一插入就让节点满了，下次插入可能分裂到自己 |
| `operation == DELETE` 且 `size > min` | 删除后还够半满 | 不会触发合并或重分配 |
| `is_root_page()` | 当前是根 | 根没有父节点可释放，一律返回 true |
| `operation == FIND` | 只读操作 | 不改变结构，一定安全 |

关键细节：isSafe 只在**非根节点**上判断 `INSERT/DELETE` 条件。根节点直接返回 true——不是因为根不会分裂，而是根的分裂由专门的 `root_latch_` 互斥锁处理（见 `find_leaf_page` 中的 `root_latch_.lock()`）。

### find_leaf_page 中的锁缩放流程

`src/index/ix_index_handle.cpp:270-318`

下面追踪"插入操作"的完整加锁流程：

```
1. root_latch_.lock()         // 先锁根互斥锁，防止根被其他线程分裂
2. 加载根节点 node
3. node->page->WLatch()       // 对根节点页面加写锁
4. if (node->isSafe(INSERT))  // 根 isSafe 总是 true
      root_latch_.unlock()    // 释放根互斥锁

5. 循环向下:
   a. child_node = fetch_node(下一级)
   b. child_node->page->WLatch()           // 先锁住孩子
   c. 把 node 加入 latch_page_set           // 记录已加锁的祖先
   d. if (child_node->isSafe(INSERT))       // 孩子安全？
        release_all_index_latch_page(txn)   // → 释放所有祖先的锁！
   e. node = child_node                     // 继续向下

6. 到达叶节点，返回
```

**关键转折**在步骤 5d：如果孩子节点 `isSafe`，说明孩子不会分裂，那祖先节点（node、node 的 parent 等）的结构都不会变——释放祖先锁，让其他线程可以访问。

如果孩子节点不安全（比如插入后会满），就**保留祖先锁**。因为孩子一旦分裂，需要修改父节点（插入新的分隔键），父节点如果之前被释放了，其他线程可能正在改它，就出问题了。

一个具体例子：

```
要插入 key=45，当前树状态：
  根节点 [40, 70]，已满（size=2, max=2）
  中间节点 [50, 62]，未满（size=2, max=3）
  目标叶节点 [40, 45, 50]，已满（size=3, max=3）

加锁过程：
1. WLock 根 → isSafe(根) = true → 释放 root_latch_
2. 下行到中间节点 [50, 62] → WLock 它 → isSafe([50,62])?
   size=2, max=3 → 2+1 < 3 = true → SAFE
   → 释放根的 WLock！
3. 下行到叶节点 [40, 45, 50] → WLock 它 → isSafe([40,45,50])?
   size=3, max=3 → 3+1 < 3 = false → NOT SAFE
   → 保留中间节点的 WLock
4. 在叶节点插入 key=45，触发分裂
5. 分裂需要修改父节点 [50, 62] → 它的锁还拿着，安全！
```

如果没有锁缩放，步骤 1~4 根和中间节点都被锁着，其他线程全堵住。

有了锁缩放，步骤 2 就释放了根锁，步骤 3 只保留中间节点的锁，并发度大幅提升。

## 源码对应

| 内容 | 文件 | 行号 |
|------|------|------|
| IxNodeHandle 完整定义 | `src/index/ix_index_handle.h` | 60-208 |
| 框架待实现方法 | `db2026-x/src/index/ix_index_handle.cpp` | 21-141 |

上一节：[02-index-data-structures.md](./02-index-data-structures.md) | 下一节：[04a-btree-search.md](./04a-btree-search.md)
