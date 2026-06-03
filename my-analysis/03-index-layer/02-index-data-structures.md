# 02. 索引层数据结构

索引层用 B+ 树组织索引键值。先看数据结构——文件存什么、页面存什么、如何定位。

## 常量

`src/index/ix_defs.h:20-25`

```cpp
constexpr int IX_NO_PAGE = -1;          // 无效页面号
constexpr int IX_FILE_HDR_PAGE = 0;     // 文件头在第 0 页
constexpr int IX_LEAF_HEADER_PAGE = 1;  // 叶头在第 1 页
constexpr int IX_INIT_ROOT_PAGE = 2;    // 初始根节点在第 2 页
constexpr int IX_INIT_NUM_PAGES = 3;    // 初始页面数
constexpr int IX_MAX_COL_LEN = 512;     // 索引列最大长度
```

索引文件初始有 3 页：第 0 页文件头、第 1 页叶头（叶节点链表哨兵）、第 2 页根节点。

## IxFileHdr：索引文件头

存在索引文件的第 0 页，存储 B+ 树的全局元信息。

`src/index/ix_defs.h:27`

```cpp
class IxFileHdr {
 public:
  page_id_t first_free_page_no_;   // 第一个空闲页面号（已删除的页面可复用）
  int num_pages_;                  // 文件总页数
  page_id_t root_page_;            // B+ 树根节点页面号
  int col_num_;                    // 索引包含的字段数
  std::vector<ColType> col_types_; // 各字段类型
  std::vector<int> col_lens_;      // 各字段长度
  int col_tot_len_;                // 所有字段总长度（单键大小）
  int btree_order_;                // B+ 树阶数（每个节点最多可插入的键值对数量）
  int keys_size_;                  // 每页 keys 区大小 = (btree_order + 1) * col_tot_len
  page_id_t first_leaf_;           // 首叶节点页号（IxScan 起点）
  page_id_t last_leaf_;            // 尾叶节点页号（IxScan 终点判断）
  int tot_len_;                    // 结构体序列化后的总长度
};
```

| 字段 | 作用 | 确定时机 |
|------|------|----------|
| `root_page_` | B+ 树入口，所有查找从此开始 | 建索引时初始化，此后可能变更 |
| `btree_order_` | 阶数，由 PAGE_SIZE 和 key 大小算出 | 建索引时计算 |
| `keys_size_` | 每页预留给 keys 的字节数 | = (order+1) × col_tot_len |
| `first_leaf_` / `last_leaf_` | 叶节点链表头尾，用于范围扫描 | 动态维护 |
| `num_pages_` | 当前文件页数，新增节点时递增 | 动态变化 |

与记录层的 `RmFileHdr` 对比：索引层多了 `root_page_`（B+ 树入口）、`btree_order_`（阶数控制分支数）、叶节点链表指针（`first_leaf_`/`last_leaf_`）。

## IxPageHdr：索引页头

每个索引页的开头，对应记录层的 `RmPageHdr`。

`src/index/ix_defs.h:140`

```cpp
class IxPageHdr {
 public:
  page_id_t next_free_page_no;  // 未使用（保留字段）
  page_id_t parent;             // 父节点页面号
  int num_key;                  // 当前节点的键值对数量
  bool is_leaf;                 // 是否为叶节点
  page_id_t prev_leaf;          // 前驱叶节点页面号（仅叶节点有效）
  page_id_t next_leaf;          // 后继叶节点页面号（仅叶节点有效）
};
```

| 字段 | 作用 |
|------|------|
| `parent` | 指向父节点，根节点的 parent = -1 |
| `num_key` | 当前节点中有几个键（= 孩子数 - 1） |
| `is_leaf` | 叶节点 / 内部节点的标志位 |
| `prev_leaf` / `next_leaf` | 叶节点双向链表，支持范围扫描 |

## Iid：索引记录标识

`src/index/ix_defs.h:153`

```cpp
class Iid {
 public:
  int page_no;  // 索引页面号
  int slot_no;  // 页面内槽位号（即 key_idx）
};
```

`Iid` 对应记录层的 `Rid`，但 `slot_no` 的含义不同：它是节点内 keys 数组的下标，指向"该键值对在节点中的位置"。

## IxNodeHandle 页面布局

索引页面内部布局比记录层简单——没有 Bitmap，只有 header + keys + rids：

```
索引页（4096 字节）：
┌─────────────────┬───────────────────┬───────────────────┐
│ IxPageHdr       │ Keys 键数组        │ Rids 孩子指针数组   │
│ sizeof(page_hdr)│ keys_size 字节     │ 剩余空间           │
└─────────────────┴───────────────────┴───────────────────┘
```

`IxNodeHandle` 的构造函数（`src/index/ix_index_handle.h:76`）：

```cpp
IxNodeHandle(const IxFileHdr* file_hdr_, Page* page_) {
  page_hdr = reinterpret_cast<IxPageHdr*>(page->get_data());
  keys = page->get_data() + sizeof(IxPageHdr);
  rids = reinterpret_cast<Rid*>(keys + file_hdr->keys_size_);
}
```

keys 数组在索引创建时就预分配了固定大小（`keys_size_`），不像记录层需要 bitmap 标记槽位。因为 B+ 树内部节点通过 `num_key` 就知道有几个键，叶节点同理。

## 源码对应

| 内容 | 文件 | 行号 |
|------|------|------|
| 常量定义 | `src/index/ix_defs.h` | 20-25 |
| IxFileHdr | `src/index/ix_defs.h` | 27-138 |
| IxPageHdr | `src/index/ix_defs.h` | 140-151 |
| Iid | `src/index/ix_defs.h` | 153-163 |
| IxNodeHandle 构造函数 | `src/index/ix_index_handle.h` | 76-81 |

下一节：[03-index-node-handle.md](./03-index-node-handle.md)
