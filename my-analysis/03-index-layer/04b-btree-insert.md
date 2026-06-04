# 04b. B+ 树插入与分裂

B+ 树插入从叶节点开始，满则分裂，分裂向上传播，直到根节点。

## 整体流程

```mermaid
flowchart TD
    A["insert_entry key, rid"] --> B["find_leaf_page<br/>找到目标叶节点"]
    B --> C["node->insert<br/>插入键值对"]
    C --> D{"node 满了?"}
    D -->|"否"| E["完成"]
    D -->|"是"| F["split 分裂节点"]
    F --> G["insert_into_parent<br/>将新节点的第一个 key 插入父节点"]
    G --> H{"父节点满了?"}
    H -->|"否"| E
    H -->|"是"| I["递归 split + insert_into_parent"]
    I --> G
```

框架中以下方法全部为空，需要自行实现。

## insert_pairs：节点内插入键值对

**含义**：在节点内部的 pos 位置插入 n 个连续键值对。最底层的插入操作，只操作当前节点，不关心树的结构变化。

**场景**：被 `insert`、`split`、`insert_into_parent`、`redistribute` 调用。

**实现**：核心是 `memmove` 腾出空间——把 pos 后面的元素右移 n 位，再把新数据 `memcpy` 填入空位。

**源码**：`src/index/ix_index_handle.cpp:142`（参考实现）

```
插入前: [A, B, D, E, _, _]  pos=2, n=1
         0  1  2  3  4  5

memmove(pos+n, pos, count-pos): 将 [D, E] 右移一位
插入后: [A, B, C, D, E, _]
         0  1  2  3  4  5
         插入 ↑ 插入C
```

```cpp
// IxNodeHandle::insert_pairs, src/index/ix_index_handle.cpp:142
void IxNodeHandle::insert_pairs(int pos, const char* key, const Rid* rid, int n) {
  auto cur_key = get_key(pos);
  auto cur_rid = get_rid(pos);
  int num_keys = page_hdr->num_key;
  int cols_len = file_hdr->col_tot_len_;

  if (pos < num_keys) {
    memmove(cur_key + n * cols_len, cur_key, (num_keys - pos) * cols_len);
    memmove(cur_rid + n, cur_rid, (num_keys - pos) * sizeof(Rid));
  }
  memcpy(cur_key, key, n * cols_len);
  memcpy(cur_rid, rid, n * sizeof(Rid));
  page_hdr->num_key += n;
}
```

## split：节点分裂

**含义**：节点已满时，从中间切分——左半留在原节点，右半搬到新节点。B+ 树长高的唯一途径。

**场景**：`insert_entry` 发现节点满了之后调用，分裂出的新键由 `insert_into_parent` 向上传播。

**源码**：`src/index/ix_index_handle.cpp:358`（参考实现）

**示例**：

```
分裂前 node: [A, B, C, D, E, F]  假设 max_size=6
                        ↓ split_point = min_size = 3
分裂后 node: [A, B, C]            new_node: [D, E, F]
```

**三步操作**：

1. 创建 `new_node`，`num_key` 初始化为 0
2. 用 `insert_pairs(0, ...)` 把 node 后半部分搬入 new_node
3. `node->set_size(split_point)` 软删除 node 的后半部分

如果是叶节点，还需要维护叶节点链表（`prev_leaf` / `next_leaf` 指针）。
如果是内部节点，新节点的孩子需要更新父指针（调用 `maintain_child`）。

## insert_into_parent：向上传播

**含义**：分裂后，将新节点的第一个 key 作为分隔键插入父节点。如果父节点也满了则递归分裂。

**场景**：`split` 之后调用。可能递归向上，最远到根节点。

**源码**：`src/index/ix_index_handle.cpp:418`（参考实现）

**两种情况**：

1. **node 是根节点**：创建新根，将 old_node 和 new_node 的第一个 key 插入新根，设置父子关系
2. **node 不是根节点**：获取父节点，在 `find_child(old_node) + 1` 位置插入 new_node 的第一个 key；如果父节点也满了，递归 `split` + `insert_into_parent`

## insert_entry：顶层插入入口

**含义**：B+ 树插入操作的入口，串联整个流程：找到位置 → 插入键值对 → 满则分裂 → 向上传播。

**场景**：执行器在插入记录后调用，同步维护索引。重复 key 会被跳过。

**源码**：`src/index/ix_index_handle.cpp:500`（参考实现）

1. `find_leaf_page(key, INSERT)` → 从根逐层下到目标叶节点，加写锁
2. `leaf_node->insert(key, value)` → 在叶节点中二分定位插入位置并插入，重复 key 则跳过
3. 节点满了 → `split` 分裂 + `insert_into_parent` 向父节点插入新分隔键（可能递归向上）
4. 如果插入位置是节点第一个 key → `maintain_parent` 向上更新父节点中对应该节点的分隔键
5. 释放写锁和 unpin 页面

## 源码对应

| 内容 | 文件 | 行号 |
|------|------|------|
| insert_pairs | `src/index/ix_index_handle.cpp` | 142-172 |
| insert | `src/index/ix_index_handle.cpp` | 181-193 |
| split | `src/index/ix_index_handle.cpp` | 358-401 |
| insert_into_parent | `src/index/ix_index_handle.cpp` | 418-468 |
| insert_entry | `src/index/ix_index_handle.cpp` | 500-561 |

上一节：[04a-btree-search.md](./04a-btree-search.md) | 下一节：[04c-btree-delete.md](./04c-btree-delete.md)
