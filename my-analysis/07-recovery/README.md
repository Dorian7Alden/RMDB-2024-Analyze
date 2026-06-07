# 第 7 章：故障恢复

故障恢复层负责在系统崩溃后通过日志把数据库恢复到一致状态，保证持久性和原子性。

## 文档列表

| 序号 | 文件 | 内容 |
|------|------|------|
| 01 | [故障恢复概述](./01-recovery-overview.md) | WAL 协议、日志类型、三步恢复算法概览 |
| 02 | [日志数据结构](./02-recovery-data-structures.md) | LogRecord 类层次、日志头布局、LogBuffer |
| 03 | [日志管理器](./03-log-manager.md) | add_log_to_buffer、flush_log_to_disk 和后台刷盘 |
| 04 | [恢复管理器](./04-recovery-manager.md) | analyze、redo、undo 三步恢复算法详解 |
| 05 | [组件交互](./05-recovery-interaction.md) | 恢复层与事务层、系统层、缓冲池的调用链 |
| 06 | [框架与参考实现对比](./06-recovery-frame-vs-reference.md) | db2026-x 中恢复模块的待实现清单 |
| 07 | [API 速查](./07-recovery-api-reference.md) | LogManager、RecoveryManager 接口速查 |
| 08 | [总结](./08-recovery-summary.md) | 第 7 章核心设计思路和学习建议 |

## 阅读顺序

建议从 01 开始按序阅读。

02 是数据结构基础，03 是日志写入路径，04 是恢复算法核心，建议连续阅读。

06 建议在读完 03、04 后再看，因为它把框架中的空方法和参考实现逐项对应起来。
