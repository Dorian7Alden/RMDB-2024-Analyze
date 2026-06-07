# 恢复层 API 速查

## 本篇目的

**含义**：本文是第 7 章故障恢复层的接口速查表。

**作用**：用于快速查阅 LogManager 和 RecoveryManager 的对外方法。

## LogManager

**位置**：`src/recovery/log_manager.h:436-489`

| 方法 | 输入 | 输出 | 副作用 |
|------|------|------|--------|
| `add_log_to_buffer(log_record)` | 日志对象指针 | 分配的 LSN | 序列化到缓冲区，缓冲区满则刷盘 |
| `flush_log_to_disk()` | 无 | 无 | 缓冲区全部写入磁盘日志文件 |
| `get_persist_lsn()` | 无 | `lsn_t` | 无 |
| `set_global_lsn(lsn)` | 新的 LSN 值 | 无 | 更新全局 LSN（恢复时使用） |
| `set_persist_lsn(lsn)` | 新的持久化 LSN | 无 | 更新持久化 LSN（恢复时使用） |

## LogRecord 层次

**基类位置**：`src/recovery/log_manager.h:36-75`

| 类 | 日志体字段 | 日志体长度 |
|----|-----------|-----------|
| `BeginLogRecord` | 无 | LOG_HEADER_SIZE |
| `CommitLogRecord` | 无 | LOG_HEADER_SIZE |
| `AbortLogRecord` | 无 | LOG_HEADER_SIZE |
| `InsertLogRecord` | `insert_value_`（RmRecord）、`rid_`（Rid）、`table_name_`（char*） | 头 + 记录大小 + Rid 大小 + 表名大小 |
| `DeleteLogRecord` | `delete_value_`（RmRecord）、`rid_`（Rid）、`table_name_`（char*） | 同上 |
| `UpdateLogRecord` | `old_value_`（RmRecord）、`update_value_`（RmRecord）、`rid_`（Rid）、`table_name_`（char*） | 头 + 旧值大小 + 新值大小 + Rid 大小 + 表名大小 |
| `StaticCheckpointLogRecord` | 无 | LOG_HEADER_SIZE |

## LogRecord 基类方法

**含义**：所有日志记录类共有的三个核心方法。

| 方法 | 作用 |
|------|------|
| `serialize(char* dest)` | 把日志记录按内存布局写入 dest 缓冲区 |
| `deserialize(const char* src)` | 从 src 缓冲区按内存布局恢复日志记录 |
| `format_print()` | 调试用打印日志内容 |

## LogBuffer

**位置**：`src/recovery/log_manager.h:419-433`

| 方法 | 输入 | 输出 | 作用 |
|------|------|------|------|
| `is_full(append_size)` | 要追加的字节数 | bool | 判断缓冲区是否有足够空间 |

## RecoveryManager

**位置**：`src/recovery/log_recovery.h:27-63`

| 方法 | 作用 | 填充的数据结构 |
|------|------|--------------|
| `analyze()` | 扫描日志文件，分析崩溃时状态 | `active_txn_`、`lsn_mapping_`、`dirty_page_table_` |
| `redo()` | 重做脏页表中的所有操作 | 修改记录和索引 |
| `undo()` | 撤销活跃事务的所有操作 | 修改记录和索引 |

## LogType 枚举

**位置**：`src/recovery/log_manager.h:22-30`

| 枚举值 | 含义 |
|--------|------|
| `UPDATE = 0` | 更新记录 |
| `INSERT = 1` | 插入记录 |
| `DELETE = 2` | 删除记录 |
| `BEGIN = 3` | 事务开始 |
| `COMMIT = 4` | 事务提交 |
| `ABORT = 5` | 事务回滚 |
| `STATIC_CHECKPOINT = 6` | 静态检查点 |

## 日志头内存偏移

**位置**：`src/recovery/log_defs.h:22-34`

| 常量 | 值 | 含义 |
|------|-----|------|
| `OFFSET_LOG_TYPE` | 0 | 日志类型偏移 |
| `OFFSET_LSN` | 4 | LSN 偏移 |
| `OFFSET_LOG_TOT_LEN` | 8 | 日志总长度偏移 |
| `OFFSET_LOG_TID` | 12 | 事务编号偏移 |
| `OFFSET_PREV_LSN` | 16 | 前一条日志 LSN 偏移 |
| `OFFSET_LOG_DATA` | 20 | 日志体起始偏移 |
| `LOG_HEADER_SIZE` | 20 | 日志头总字节数 |

上一节：[06-recovery-frame-vs-reference.md](./06-recovery-frame-vs-reference.md) | 下一节：[08-recovery-summary.md](./08-recovery-summary.md)
