# 第 4 章：系统管理

系统管理层（SM = System Manager）负责数据库的**元数据管理**和 **DDL 执行**。

## 文档列表

| 序号 | 文件 | 内容 |
|------|------|------|
| 01 | [系统管理概述](./01-system-layer-overview.md) | SM 的定位、职责、架构位置 |
| 02 | [数据结构](./02-system-data-structures.md) | ColMeta、IndexMeta、TabMeta、DbMeta 层次结构 |
| 03 | [数据库操作](./03-database-operations.md) | create/open/close/drop 数据库 + flush_meta |
| 04 | [表操作](./04-table-operations.md) | create/drop table + show_tables / desc_table |
| 05 | [索引操作](./05-index-operations.md) | create/drop/redo index 概览 |
| 05b | [创建索引详解](./05b-create-index-detail.md) | 全表扫描 + memcpy 拼键 + B+ 树插入 |
| 06 | [组件交互](./06-system-interaction.md) | SM 与 RM/IX/执行器的调用关系 |
| 07 | [框架与参考实现对比](./07-system-frame-vs-reference.md) | 逐项对比，分析差异原因 |
| 08 | [API 速查](./08-system-api-reference.md) | SmManager 全部方法 + 元数据结构字段一览 |
| 09 | [总结](./09-system-summary.md) | 框架状态表、核心设计思路、与前三章的联系 |

## 阅读顺序

建议从 01 开始按序号顺序阅读。05b（创建索引详解）是 SM 层最复杂的操作，建议在理解 05 的概览后再深入。
