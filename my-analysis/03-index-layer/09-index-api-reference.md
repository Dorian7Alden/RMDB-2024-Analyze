# 09. 索引层 API 速查

## IxManager

`src/index/ix_manager.h:21`

```cpp
void create_index(filename, col_names, col_types, col_lens)  // 创建索引文件
void open_index(filename) → IxIndexHandle*                    // 打开索引文件
void close_index(IxIndexHandle*)                               // 关闭索引文件
void destroy_index(filename)                                   // 删除索引文件
```

## IxIndexHandle

B+ 树操作枢纽。`src/index/ix_index_handle.h:211`

```cpp
// 查找
bool get_value(key, result, txn) → bool                   // 查找 key，返回 Rid 列表
Iid lower_bound(key) → Iid                                // 查找下界
Iid upper_bound(key) → Iid                                // 查找上界
Iid leaf_begin() → Iid                                    // 扫描起点
Iid leaf_end() → Iid                                      // 扫描终点后一个

// 插入
page_id_t insert_entry(key, rid, txn)                     // 插入键值对
IxNodeHandle* split(node) → IxNodeHandle*                 // 分裂节点
void insert_into_parent(old_node, key, new_node, txn)     // 向上插入

// 删除
bool delete_entry(key, txn)                               // 删除键值对
bool coalesce_or_redistribute(node, txn, root_latched)    // 下溢处理
bool adjust_root(old_root)                                // 根调整
void redistribute(neighbor, node, parent, index)          // 重分配
bool coalesce(neighbor, node, parent, index, txn, root)   // 合并

// 页面管理
IxNodeHandle* fetch_node(page_no)                         // 获取已有节点
IxNodeHandle* create_node()                               // 创建新节点
void maintain_parent(node)                                // 向上更新父节点的第一个 key
void maintain_child(node, child_idx)                      // 维护子节点父指针
```

## IxNodeHandle

单个 B+ 树节点。`src/index/ix_index_handle.h:60`

```cpp
// 查找（需实现）
int lower_bound(target)                                   // 二分查找 >= target
int upper_bound(target)                                   // 二分查找 > target
bool leaf_lookup(key, &value)                             // 叶节点精确查找
page_id_t internal_lookup(key)                            // 内部节点找孩子

// 插入删除（需实现）
void insert_pairs(pos, key, rid, n)                       // pos 处插入 n 个键值对
int insert(key, rid) → int                                // 单键插入
void erase_pair(pos)                                      // 删除单个键值对
int remove(key) → int                                     // 查找并删除

// 属性查询（框架已实现）
int get_size() / set_size() / get_max_size() / get_min_size()
char* get_key(idx) / void set_key(idx, key)
Rid* get_rid(idx) / void set_rid(idx, rid)
bool is_leaf_page() / is_internal_page() / is_root_page()
page_id_t get_next_leaf() / get_prev_leaf()
bool isSafe(operation)                                    // 锁缩放判断
```

## IxScan

叶节点链表遍历。`src/index/ix_scan.h:21`

```cpp
void next()                                                // 移动到下一个记录
bool is_end()                                              // 到达 end
Rid rid()                                                  // 当前 Rid
RmRecord get_key()                                         // 当前键
```

## 数据结构

```cpp
// IxFileHdr  src/index/ix_defs.h:27
class IxFileHdr { first_free_page_no_; num_pages_; root_page_;
  col_num_; col_types_; col_lens_; col_tot_len_;
  btree_order_; keys_size_; first_leaf_; last_leaf_; tot_len_; }

// IxPageHdr  src/index/ix_defs.h:140
class IxPageHdr { next_free_page_no; parent; num_key;
  is_leaf; prev_leaf; next_leaf; }

// Iid  src/index/ix_defs.h:153
class Iid { int page_no; int slot_no; }
```

上一节：[08-index-frame-vs-reference.md](./08-index-frame-vs-reference.md) | 下一节：[10-index-summary.md](./10-index-summary.md)
