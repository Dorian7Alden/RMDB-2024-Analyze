# 第 6 章：事务与并发控制

事务与并发控制层负责保证多条 SQL 同时执行时仍然像按某个顺序串行执行一样安全。

## 文档列表

| 序号 | 文件 | 内容 |
|------|------|------|
| 01 | [事务与并发概述](./01-transaction-concurrency-overview.md) | 事务层的位置、核心概念和生命周期 |
| 02 | [事务数据结构](./02-transaction-data-structures.md) | Transaction、WriteRecord、LockDataId、LockRequestQueue 等结构 |
| 03 | [事务管理器](./03-transaction-manager.md) | begin、commit、abort 和写集回滚 |
| 04 | [锁管理器](./04-lock-manager.md) | 两阶段封锁、多粒度锁、锁兼容和死锁预防 |
| 05 | [间隙锁](./05-gap-lock.md) | Gap、间隙锁表和幻读保护 |
| 06 | [组件交互](./06-transaction-interaction.md) | 执行层、记录层、索引层、事务层的调用关系 |
| 07 | [框架与参考实现对比](./07-transaction-frame-vs-reference.md) | db2026-x 中事务并发模块的待实现清单 |
| 08 | [API 速查](./08-transaction-api-reference.md) | TransactionManager、LockManager 和事务结构接口速查 |
| 09 | [总结](./09-transaction-summary.md) | 第 6 章核心设计思路和学习建议 |

## 阅读顺序

建议从 01 开始按序阅读。

02 是数据结构基础，03 和 04 是实现主线，05 是可串行化隔离中防幻读的扩展内容。

07 建议在读完 03、04、05 后再看，因为它把框架中的空方法和参考实现逐项对应起来。
