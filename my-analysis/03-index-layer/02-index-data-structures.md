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

## B+ 树磁盘存储全景

插入一些数据后，`student.idx` 文件在磁盘上的样子：

```mermaid
flowchart TD
    subgraph L0[" "]
        style L0 fill:none,stroke:none
        HDR["P0 文件头 IxFileHdr<br/>root_page=2<br/>first_leaf=6 last_leaf=14<br/>num_pages=15"]
    end

    subgraph L1[" "]
        style L1 fill:none,stroke:none
        LH["P1 叶头哨兵"] & R["P2 根节点<br/>keys=[40,70]"]
    end

    subgraph L2["内部节点层"]
        style L2 fill:#dbeafe,stroke:#3b82f6,color:#1e40af
        I3["P3<br/>keys=[15,28]"] & I4["P4<br/>keys=[50,62]"] & I5["P5<br/>keys=[80,92]"]
    end

    subgraph L3["叶节点层 双向链表"]
        style L3 fill:#d1fae5,stroke:#10b981,color:#065f46
        L6["P6<br/>keys=[5,10,15]"] --> L7["P7<br/>keys=[20,24,28]"] --> L8["P8<br/>keys=[32,38]"] --> L9["P9<br/>keys=[40,45,50]"] --> L10["P10<br/>keys=[55,62]"] --> L11["P11<br/>keys=[66,70]"] --> L12["P12<br/>keys=[72,78,80]"] --> L13["P13<br/>keys=[86,92]"] --> L14["P14<br/>keys=[96,99]"]
    end

    HDR -->|"root_page"| R
    HDR -->|"first_leaf"| L6
    HDR -->|"last_leaf"| L14
    R -->|"rids[0]"| I3
    R -->|"rids[1]"| I4
    R -->|"rids[2]"| I5
    I3 -->|"rids[0]"| L6
    I3 -->|"rids[1]"| L7
    I3 -->|"rids[2]"| L8
    I4 -->|"rids[0]"| L9
    I4 -->|"rids[1]"| L10
    I4 -->|"rids[2]"| L11
    I5 -->|"rids[0]"| L12
    I5 -->|"rids[1]"| L13
    I5 -->|"rids[2]"| L14

    classDef header fill:#fef3c7,stroke:#f59e0b,color:#92400e
    classDef sentinel fill:#e0e0e0,stroke:#9e9e9e,color:#616161
    class HDR header
    class LH sentinel
```

**从上图可以看出**：
- 一棵 3 层的 B+ 树：根（1 个）→ 内部节点（3 个）→ 叶节点（9 个），共 15 页
- 内部节点（蓝底）只存分隔键和孩子指针，不存实际数据
- 叶节点（绿底）存实际键值和记录 Rid，所有叶节点通过 `prev_leaf`/`next_leaf` 串成一条双向链表
- 范围扫描只需沿链表顺序遍历，不需要回溯内部节点
- 每个节点的 `parent` 字段指回父节点，根节点的 `parent = -1`

**每个页面内部**都是三段式布局：

```
页面内部（4096 字节）：
┌──────────────────┬───────────────────────────┬──────────────────────────┐
│ IxPageHdr        │ keys[0..btree_order]      │ rids[0..btree_order]     │
│ parent num_key   │ 键值数组，keys_size 字节    │ 孩子指针数组               │
│ is_leaf          │ col_tot_len × (order+1)   │ sizeof(Rid) × (order+1)  │
│ prev next        │                           │                          │
└──────────────────┴───────────────────────────┴──────────────────────────┘
```

keys 和 rids 在创建时就预分配了固定大小（由 `btree_order` 决定），不随数据量动态变化。

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

## 源码对应

| 内容 | 文件 | 行号 |
|------|------|------|
| 常量定义 | `src/index/ix_defs.h` | 20-25 |
| IxFileHdr | `src/index/ix_defs.h` | 27-138 |
| IxPageHdr | `src/index/ix_defs.h` | 140-151 |
| Iid | `src/index/ix_defs.h` | 153-163 |
| IxNodeHandle 构造函数 | `src/index/ix_index_handle.h` | 76-81 |

下一节：[03-index-node-handle.md](./03-index-node-handle.md)
