# 故障恢复总结

## 本章覆盖范围

**含义**：第 7 章讲的是数据库在崩溃后如何恢复到一致性状态。

**范围**：本章覆盖 WAL 协议、日志记录结构、LogManager 日志写入和 RecoveryManager 三步恢复算法。

## 核心数据流

**含义**：恢复的本质是通过日志记录"做过什么"，崩溃后通过日志重做或撤销。

| 组件 | 角色 | 核心数据结构 |
|------|------|-------------|
| LogRecord 层次 | 每种操作的持久化表示 | log_type_、lsn_、prev_lsn_、操作数据 |
| LogManager | 日志写入和刷盘管理 | log_buffer_、global_lsn_、persist_lsn_ |
| RecoveryManager | 崩溃恢复执行者 | active_txn_、lsn_mapping_、dirty_page_table_ |
| DiskManager | 物理日志文件读写 | write_log、read_log |

## 和前六章的关系

**含义**：恢复层是事务层持久性的实现者，依赖于前面所有层。

| 已学章节 | 本章如何使用 |
|----------|--------------|
| 存储层 | LogManager 通过 DiskManager 读写日志文件 |
| 记录层 | redo 和 undo 调用 RmFileHandle 的 insert、delete、update |
| 索引层 | redo 和 undo 调用 IxIndexHandle 的 insert_entry、delete_entry |
| 系统层 | RecoveryManager 持有 SmManager 指针获取表和索引元数据 |
| 事务层 | RecoveryManager 通过 TransactionManager 恢复事务编号；日志和写集记录相同内容 |
| 缓冲池 | analyze 阶段检查 Page LSN 判断是否需要重做 |

## 框架学习重点

**含义**：`db2026-x/` 中恢复层的待实现内容分日志类和业务方法两组。

| 优先级 | 项目 | 原因 |
|--------|------|------|
| 1 | CommitLogRecord、AbortLogRecord | 最简单的日志类，只有头部无日志体 |
| 2 | DeleteLogRecord、UpdateLogRecord | 需要序列化 RmRecord、Rid 和表名 |
| 3 | add_log_to_buffer | 日志写入路径的核心，依赖日志类的 serialize |
| 4 | flush_log_to_disk | 日志持久化的最后一步 |
| 5 | analyze | 依赖所有日志类的 deserialize，最复杂的扫描与分类逻辑 |
| 6 | redo | 依赖 analyze 的输出，需要调用记录层和索引层 |
| 7 | undo | 最复杂的撤销逻辑，需要大根堆跨事务逆序撤销 |

**建议**：先完成全部日志类（第 1 到 4 项），跑通正常运行时的日志写入；再实现 analyze 确保日志扫描正确；最后实现 redo 和 undo 并通过崩溃恢复测试。

## 关键设计思路

### WAL

**含义**：WAL（Write-Ahead Logging，预写日志）要求先写日志再改数据。

**作用**：它保证崩溃时总可以通过日志重建数据的最新状态。

**示例**：如果先改数据页再写日志，在改完数据但日志没写时崩溃，恢复时不知道这条数据是被哪个事务修改的、是应该保留还是回滚。

### 三步恢复算法

**含义**：analyze-redo-undo 的三步逻辑来自 ARIES 恢复算法。

**作用**：analyze 确定范围，redo 把已提交的改完，undo 把未提交的撤回。

**示例**：T1 已提交两条 INSERT，T2 未提交一条 DELETE——analyze 找出 T2 是活跃的、两条 INSERT 对应的页面需要 redo；redo 重做两条 INSERT；undo 撤销 T2 的 DELETE 把它插回去。

### LSN 链与逆序撤销

**含义**：每条日志的 `prev_lsn_` 形成事务内部的日志链。

**作用**：undo 沿着这条链从后往前撤销，保证撤销顺序与原始执行顺序相反。

**示例**：T3 先 INSERT 后 UPDATE——如果先撤销 INSERT，记录被删了，后续 UPDATE 的撤销找不到记录；正确做法是逆序——先撤销 UPDATE 恢复旧值，再撤销 INSERT 删除记录。

### 页 LSN 与脏页判断

**含义**：每个数据页上记录该页最近一次修改对应的日志 LSN。

**作用**：analyze 阶段对比页 LSN 和日志 LSN，决定该日志的操作是否需要 redo——页 LSN 更旧说明修改没落盘。

**示例**：某条 INSERT 日志 lsn=10，对应数据页上的 LSN 是 5，说明这条插入的数据还没写入数据页，redo 时需要重新插入。

### 日志与写集的同构

**含义**：事务写集和日志记录的内容完全相同，一个用于运行时回滚，一个用于崩溃恢复。

**作用**：这保证了 abort 和崩溃恢复 undo 的行为一致性——同样的修改信息，同样的反向操作。

| 恢复方式 | 信息源 | 触发时机 |
|---------|--------|---------|
| 运行时回滚 | Transaction::write_set_ | 事务异常或显式 ROLLBACK |
| 崩溃恢复 undo | RecoveryManager + 日志文件 | 系统重启 |

## 常见误区

| 误区 | 正确认识 |
|------|----------|
| 日志和写集是两种不同数据 | 内容完全一致，只是用途和生命周期不同 |
| 恢复只需要回滚未提交的事务 | 也需要重做已提交但未落盘的修改 |
| flush_log_to_disk 每次加日志都调用 | 实际只在缓冲区满和后台定时时调用，减少磁盘 I/O |
| redo 按 LSN 从大到小处理 | redo 从前往后（从小到大），undo 从后往前（从大到小） |
| 恢复后 global_lsn_ 从 0 开始 | 必须恢复到崩溃前最大值加 1，否则 LSN 会重复 |

## 本章完成后应该掌握什么

**能力 1**：能解释 WAL 的含义和为什么必须先写日志再改数据。

**能力 2**：能画出日志头在内存中的字节布局，并解释五个字段的作用。

**能力 3**：能说清楚 analyze、redo、undo 三个阶段各自的输入、输出和处理规则。

**能力 4**：能根据框架空方法的位置找到参考实现并完成补全。

**能力 5**：能理解日志和事务写集的关系——同样的数据，不同的触发路径。

## 全教程回顾

**含义**：经过全部 7 章的学习，RMDB 的分层架构已经讲完。

| 章节 | 模块 | 核心内容 |
|------|------|---------|
| 第 1 章 | 存储层 | 磁盘文件、页面、缓冲池、LRU 替换 |
| 第 2 章 | 记录层 | 记录布局、bitmap、CRUD、扫描 |
| 第 3 章 | 索引层 | B+ 树搜索、插入、删除、扫描 |
| 第 4 章 | 系统层 | 数据库管理、表管理、索引管理、元数据 |
| 第 5 章 | 查询处理 | Parser、Analyze、Optimizer、Executor |
| 第 6 章 | 事务与并发 | 两阶段锁定、多粒度锁、间隙锁、死锁预防 |
| 第 7 章 | 故障恢复 | WAL、日志管理、三步恢复算法 |

**总结**：一个 SQL 请求从网络到达 RMDB，经过 Parser（语法）、Analyze（语义）、Optimizer（计划）、Executor（执行），在执行过程中通过 BufferPool（缓存）、Record（记录）、Index（索引）读写页面，通过 LockManager（锁）和 Transaction（事务）保证并发安全，通过 LogManager（日志）保证持久性——这就是一个完整 DBMS 的运转方式。

**延伸了解：标准 ARIES 的高级特性**：RMDB 实现的是简化版 ARIES。完整的 ARIES 还包括模糊检查点（Fuzzy Checkpoint）——检查点时不需要 flush 全部脏页，不阻塞事务，只记录活跃事务表和脏页表即可。以及组提交（Group Commit）——多个事务的日志合并成一次 fsync，大幅提升写事务吞吐量。RMDB 的 1 秒间隔后台刷写线程是组提交的雏形——它天然会把 1 秒内的多条日志合并写入。

上一节：[07-recovery-api-reference.md](./07-recovery-api-reference.md)
