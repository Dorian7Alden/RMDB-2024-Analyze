# 10. 索引层总结

索引层共 11 个小节，覆盖了 B+ 树的完整实现链路：

```
IxManager → IxIndexHandle → IxNodeHandle → IxScan
(文件管理)  (B+树操作)     (节点操作)    (范围扫描)
```

## 各模块框架状态与学习要点

| 模块 | 框架状态 | 核心学习点 |
|------|---------|-----------|
| 数据结构定义 | 已完整实现 | IxFileHdr 元信息、IxPageHdr 节点信息、Iid 定位 |
| IxNodeHandle 辅助 | 已完整实现 | get_key/get_rid/set 系列、size 管理 |
| 节点内查找 | **需实现** | lower_bound/upper_bound 二分查找 |
| 节点内插入删除 | **需实现** | insert_pairs/erase_pair memmove 操作 |
| B+ 树查找 | **需实现** | find_leaf_page 锁缩放、get_value |
| B+ 树插入 | **需实现** | insert_entry、split、insert_into_parent 递归 |
| B+ 树删除 | **需实现** | delete_entry、coalesce、redistribute |
| IxManager | 已完整实现 | 索引文件创建/打开/关闭 |
| IxScan | 基本完整 | 叶节点链表遍历 |

## 核心设计思路

1. **B+ 树结构**：内部节点存分隔键+孩子指针，叶节点存实际键+数据 Rid
2. **叶节点链表**：所有叶节点通过 prev_leaf/next_leaf 串联，支持范围扫描
3. **分裂与合并**：插入满则分裂向上传播，删除空则合并或重分配
4. **锁缩放**：向下遍历时，如果子节点安全则释放父节点锁

## 与记录层的配合

索引层返回 `Rid`（记录位置），上层用 `Rid` 去记录层取实际数据。
索引层和记录层通过 `Rid` 解耦——索引只管"定位"，记录层管"取数据"。

上一节：[09-index-api-reference.md](./09-index-api-reference.md)

下一章：[第 4 章：系统管理](../04-system-layer/README.md)（待编写）
