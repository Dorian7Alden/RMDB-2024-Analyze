# 事务层 API 速查

## 本篇目的

**含义**：本文是第 6 章事务与并发控制层的接口速查表。

**作用**：用于快速查阅 Transaction、TransactionManager、LockManager 的对外方法和参数。

## TransactionManager

**位置**：`src/transaction/transaction_manager.h:23-101`

| 方法 | 输入 | 输出 | 副作用 |
|------|------|------|--------|
| `begin(txn, log_manager)` | txn 可为空指针，非空则复用已有事务 | 事务对象指针 | 写入全局事务表，写 BEGIN 日志 |
| `commit(txn, log_manager)` | 要提交的事务和日志管理器 | 无 | 清理写集，释放所有锁，写 COMMIT 日志，状态改 COMMITTED |
| `abort(txn, log_manager)` | 要回滚的事务和日志管理器 | 无 | 逆序撤销写集，释放所有锁，写 ABORT 日志，状态改 ABORTED |
| `get_transaction(txn_id)` | 事务编号 | 事务对象指针或空指针 | 如果已结束则删除对象并移除 |

## LockManager

**位置**：`src/transaction/concurrency/lock_manager.h:69-90`

| 方法 | 输入 | 输出 | 锁资源 |
|------|------|------|--------|
| `lock_shared_on_record(txn, rid, tab_fd)` | 事务、记录标识、表 fd | bool | 单条记录 |
| `lock_exclusive_on_record(txn, rid, tab_fd)` | 事务、记录标识、表 fd | bool | 单条记录 |
| `lock_shared_on_table(txn, tab_fd)` | 事务、表 fd | bool | 整张表 |
| `lock_exclusive_on_table(txn, tab_fd)` | 事务、表 fd | bool | 整张表 |
| `lock_IS_on_table(txn, tab_fd)` | 事务、表 fd | bool | 整张表 |
| `lock_IX_on_table(txn, tab_fd)` | 事务、表 fd | bool | 整张表 |
| `lock_shared_on_gap(txn, index_meta, gap, tab_fd)` | 事务、索引元数据、范围、表 fd | bool | 索引间隙 |
| `lock_exclusive_on_gap(txn, index_meta, gap, tab_fd)` | 事务、索引元数据、范围、表 fd | bool | 索引间隙 |
| `isSafeInGap(txn, index_meta, record, tab_fd)` | 事务、索引元数据、待插入记录、表 fd | bool | 检查索引间隙 |
| `unlock(txn, lock_data_id)` | 事务、锁资源 | bool | 释放单个锁 |

## Transaction

**位置**：`src/transaction/transaction.h:21-99`

| 方法组 | 方法 | 作用 |
|--------|------|------|
| 标识 | `get_transaction_id()` | 获取事务编号 |
| 标识 | `get_thread_id()` | 获取关联线程 |
| 状态 | `get_state()` / `set_state(state)` | 设置事务状态 |
| 隔离 | `get_isolation_level()` | 获取隔离级别 |
| 时间戳 | `set_start_ts(ts)` / `get_start_ts()` | 读写事务开始时间戳 |
| 日志 | `set_prev_lsn(lsn)` / `get_prev_lsn()` | 读写最后一条日志 LSN |
| 写集 | `get_write_set()` | 获取写集指针 |
| 写集 | `append_write_record(wr)` | 追加写操作记录 |
| 锁 | `get_lock_set()` | 获取已持有锁的集合指针 |
| 索引 | `get_index_latch_page_set()` | 获取事务持有的索引页面集合 |
| 索引 | `append_index_latch_page_set(page)` | 追加索引页面 |
| 索引 | `get_index_deleted_page_set()` | 获取事务删除的索引页面集合 |
| 索引 | `append_index_deleted_page(page)` | 追加已删除的索引页面 |

## TransactionState

**位置**：`src/transaction/txn_defs.h:23-24`

| 枚举值 | 含义 |
|--------|------|
| `DEFAULT` | 事务刚创建，没做任何事 |
| `GROWING` | 两阶段锁第一阶段，正在加锁 |
| `SHRINKING` | 两阶段锁第二阶段，已释放过锁 |
| `COMMITTED` | 事务已提交 |
| `ABORTED` | 事务已回滚 |

## LockMode 与 GroupLockMode

**位置**：`src/transaction/concurrency/lock_manager.h:22-34`

| LockMode | 含义 | 对应 GroupLockMode |
|----------|------|-------------------|
| `INTENTION_SHARED` | 意向共享锁 | IS |
| `INTENTION_EXCLUSIVE` | 意向排他锁 | IX |
| `SHARED` | 共享锁 | S |
| `S_IX` | 共享加意向排他 | SIX |
| `EXCLUSIVE` | 排他锁 | X |

**含义**：`LockMode` 是单个事务请求的锁类型，`GroupLockMode` 是整个队列上当前排他性最强的级别。

## AbortReason 与 TransactionAbortException

**位置**：`src/transaction/txn_defs.h:353-392`

| AbortReason | 触发条件 |
|-------------|---------|
| `LOCK_ON_SHIRINKING` | 事务在 SHRINKING 阶段尝试加锁 |
| `UPGRADE_CONFLICT` | 锁升级过程中发现冲突 |
| `DEADLOCK_PREVENTION` | wait-die 策略下年轻事务被老事务阻塞 |

**含义**：`TransactionAbortException` 是事务需要中止时抛出的异常，`rmdb.cpp` 的顶层 catch 会调用 `abort`。

上一节：[07-transaction-frame-vs-reference.md](./07-transaction-frame-vs-reference.md) | 下一节：[09-transaction-summary.md](./09-transaction-summary.md)
