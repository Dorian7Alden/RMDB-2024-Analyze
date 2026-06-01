---
name: project-overview
description: RMDB 项目的模块结构概览，帮助理解各模块定位
metadata: 
  node_type: memory
  type: project
  originSessionId: f09191ab-bbf7-4940-9fd0-e5fb21be86df
---

## RMDB 项目模块结构

```
src/
├── storage/        # 存储层 - 缓冲池管理、磁盘管理、页面管理
├── record/         # 记录层 - 记录管理（RM = Record Manager）
├── index/          # 索引层 - 索引管理（IX = IndeX）
├── system/         # 系统层 - 系统管理（SM = System Manager）
├── transaction/    # 事务层 - 事务管理、并发控制
├── execution/      # 执行层 - 查询执行器（各类算子）
├── optimizer/      # 优化器 - 查询优化与计划生成
├── parser/         # 解析器 - SQL 解析
├── analyze/        # 分析器 - 语义分析
├── replacer/       # 替换器 - LRU 页面替换算法
├── recovery/       # 恢复层 - 检查点与日志恢复
├── common/         # 公共模块 - 配置
└── tests/          # 单元测试
```

Why: 用户需要从顶层理解项目结构，后续教程会按模块逐个深入讲解。
How to apply: 在介绍任何模块时，先说明它在这个整体架构中的位置。
