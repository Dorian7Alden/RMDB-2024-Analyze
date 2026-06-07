# 08. 框架与参考实现对比

索引层的框架代码只提供了骨架——数据结构定义和辅助方法完整，但 B+ 树核心算法全部为空。

## 差异总览

| 方面 | 框架 `db2026-x/` | 参考实现 `src/` |
|------|-----------------|-----------------|
| 数据结构（IxFileHdr 等） | 完整 | 同框架 |
| IxNodeHandle 辅助方法 | 完整（get/set 系列） | 同框架 |
| 节点内查找（lower_bound） | 空，返回 -1 | 二分查找 |
| 叶节点查找（leaf_lookup） | 空，返回 false | lower_bound + 比较 |
| 内部节点查找（internal_lookup） | 空 | upper_bound - 1 |
| B+ 树遍历（find_leaf_page） | 空，返回 nullptr | 完整实现，其中锁缩放由 isSafe 实现 |
| 节点内插入（insert_pairs） | 空 | memmove + memcpy |
| 节点内删除（erase_pair） | 空 | memmove 左移 |
| B+ 树插入（insert_entry） | 空，返回 -1 | find + insert + split |
| 节点分裂（split） | 空，返回 nullptr | 创建新节点 + 迁移数据 |
| 向上插入（insert_into_parent） | 空 | 递归分裂 |
| B+ 树删除（delete_entry） | 空，返回 false | find + remove + coalesce |
| 重分配（redistribute） | 空 | 从兄弟借一个键 |
| 合并（coalesce） | 空 | 全量迁移到左兄弟 |
| 下溢处理（coalesce_or_redistribute） | 空 | 核心决策逻辑 |
| 根节点调整（adjust_root） | 空 | 提升孩子/重置根 |
| IxManager | 完整 | 同框架 |
| IxScan | 基本完整 | 同框架 |

## 框架提供了哪些

框架已经给好了以下内容，不需要改：

- 数据结构和常量定义
- IxNodeHandle 的 get/set/判断 等辅助方法
- fetch_node / create_node
- maintain_parent / maintain_child / erase_leaf
- leaf_begin / leaf_end / get_rid

## 需要自行实现的核心

**必须实现的（B+ 树核心算法）**：
1. `lower_bound` / `upper_bound` — 节点内二分查找
2. `leaf_lookup` / `internal_lookup` — 节点内查找
3. `insert_pairs` — 节点内插入
4. `erase_pair` — 节点内删除
5. `find_leaf_page` — 从根到叶的遍历
6. `split` — 节点分裂
7. `insert_into_parent` — 向上传播分裂
8. `insert_entry` — 顶层插入
9. `delete_entry` — 顶层删除
10. `adjust_root` — 根节点调整
11. `redistribute` / `coalesce` — 下溢处理

**建议实现的（增强功能）**：
12. `isSafe` — 锁缩放判断
13. 范围查询的 `lower_bound` / `upper_bound`（树级 Iid 版本）

## 学习建议

这是整个项目中最复杂的部分。建议按顺序实现：先做节点内操作（查找、插入、删除），再做树级操作（find_leaf_page、insert_entry、delete_entry）。每实现一个就写单元测试验证。

上一节：[07-index-interaction.md](./07-index-interaction.md) | 下一节：[09-index-api-reference.md](./09-index-api-reference.md)
