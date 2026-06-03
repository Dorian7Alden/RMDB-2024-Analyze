# 第 3 章：索引层

索引层（Index Manager，简称 IX）实现基于 B+ 树的索引，支持对表中某一列或多列建立索引，加速查询。

## 框架 vs 参考实现

| 模块 | 框架 `db2026-x/` | 参考实现 `src/` | 学习目标 |
|------|-----------------|-----------------|----------|
| 数据结构定义 | 已给完整定义 | 同框架（无变化） | 了解 |
| IxNodeHandle | 已给基础实现 | 补充了查找等操作 | 掌握 |
| IxIndexHandle 查找 | 已给基础实现 | 完善了并发控制 | 掌握 |
| B+ 树插入/分裂 | **需从零实现** | 完整实现 | **核心学习目标** |
| B+ 树删除/合并 | **需从零实现** | 完整实现 | **核心学习目标** |
| IxManager | 已完整实现 | 同框架（无变化） | 了解 |
| IxScan | 已给基础实现 | 优化了范围扫描 | **核心学习目标** |

## 本章目录

| 序号 | 文档 | 内容 | 类型 |
|------|------|------|------|
| 01 | [索引层概述](./01-index-layer-overview.md) | 索引层是什么、架构位置、B+ 树简介 | 理解 |
| 02 | [数据结构](./02-index-data-structures.md) | IxFileHdr、IxPageHdr、Iid、页面布局 | 掌握 |
| 03 | [IxNodeHandle](./03-index-node-handle.md) | 节点级读写、查找、isSafe | 掌握 |
| 04a | [B+ 树查找](./04a-btree-search.md) | find_leaf_page、internal_lookup、leaf_lookup | **核心** |
| 04b | [B+ 树插入](./04b-btree-insert.md) | insert_entry、split、insert_into_parent | **核心** |
| 04c | [B+ 树删除](./04c-btree-delete.md) | delete_entry、coalesce、redistribute | **核心** |
| 05 | [IxManager](./05-index-manager.md) | 索引文件生命周期管理 | 了解 |
| 06 | [IxScan](./06-index-scan.md) | 叶节点链表范围扫描 | **核心** |
| 07 | [组件交互](./07-index-interaction.md) | 各组件调用关系、索引与记录层配合 | 理解 |
| 08 | [框架对比](./08-index-frame-vs-reference.md) | 框架待实现清单与学习建议 | 理解 |
| 09 | [API 速查](./09-index-api-reference.md) | 所有类和接口的方法签名速查 | 速查 |
| 10 | [总结](./10-index-summary.md) | 各模块框架状态与核心学习点汇总 | 总结 |
