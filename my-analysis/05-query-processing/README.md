# 第 5 章：查询处理

查询处理层负责把 SQL 文本转换成可以执行的算子树，并驱动算子产生查询结果。

## 文档列表

| 序号 | 文件 | 内容 |
|------|------|------|
| 01 | [查询处理概述](./01-query-processing-overview.md) | SQL 从文本到结果的四阶段流水线 |
| 02 | [解析器](./02-parser-detail.md) | Parser 的输入输出、词法语法分析和 AST 结构 |
| 03a | [分析器数据结构](./03a-analyze-data-structures.md) | Query、Condition、TabCol、Value 等核心结构 |
| 03b | [分析器处理流程](./03b-analyze-processing-flow.md) | do_analyze 的 SELECT、INSERT、DELETE、UPDATE 处理 |
| 04 | [优化器](./04-optimizer-detail.md) | Optimizer、Planner、Plan 树和索引选择 |
| 05 | [执行层](./05-execution-detail.md) | Executor、Volcano 模型和各类算子 |
| 06 | [组件交互](./06-query-processing-interaction.md) | rmdb、Analyze、Optimizer、Portal、QlManager、Executor 的调用链 |
| 07 | [框架与参考实现对比](./07-query-processing-frame-vs-reference.md) | db2026-x 中查询处理模块的待实现清单 |
| 08 | [API 速查](./08-query-processing-api-reference.md) | Query、Plan、Executor、Portal、QlManager 接口速查 |
| 09 | [总结](./09-query-processing-summary.md) | 第 5 章核心设计思路和学习建议 |

## 阅读顺序

建议从 01 开始按序阅读。

03a 和 03b 是 Analyze (分析器) 的核心，04 是 Optimizer (优化器) 的核心，05 是 Execution (执行层) 的核心。

07 建议在读完 03b、04、05 后再看，因为它把框架中的 TODO 和参考实现逐项对应起来。
