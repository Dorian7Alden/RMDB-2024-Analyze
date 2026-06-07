# 锁管理器

## 基本职责

**含义**：`LockManager` 是事务级锁的集中管理器。

**作用**：它实现 Two Phase Locking（两阶段封锁，简称 2PL），通过表锁、记录锁和意向锁控制并发读写。

**场景**：执行层的扫描算子和修改算子会在访问记录前调用 LockManager 申请锁。

```cpp
// src/transaction/concurrency/lock_manager.h:69-90
bool lock_shared_on_gap(Transaction* txn, IndexMeta& index_meta, Gap& gap,
                        int tab_fd);

bool lock_exclusive_on_gap(Transaction* txn, IndexMeta& index_meta, Gap& gap,
                           int tab_fd);

bool lock_shared_on_record(Transaction* txn, const Rid& rid, int tab_fd);

bool lock_exclusive_on_record(Transaction* txn, const Rid& rid, int tab_fd);

bool lock_shared_on_table(Transaction* txn, int tab_fd);

bool lock_exclusive_on_table(Transaction* txn, int tab_fd);

bool lock_IS_on_table(Transaction* txn, int tab_fd);

bool lock_IX_on_table(Transaction* txn, int tab_fd);

bool unlock(Transaction* txn, const LockDataId& lock_data_id);
```

**示例**：顺序扫描读取一张表时申请表级 S 锁，插入记录时申请表级 X 锁或索引间隙 X 锁。

## 两阶段封锁检查

**含义**：`check_lock` 是所有加锁入口前的状态检查。

**作用**：它保证事务释放锁后不能再申请新锁。

```cpp
// src/transaction/concurrency/lock_manager.cpp:14-32
static inline bool check_lock(Transaction* txn) {
  auto& txn_state = txn->get_state();
  if (txn_state == TransactionState::COMMITTED ||
      txn_state == TransactionState::ABORTED) {
    return false;
  }
  if (txn_state == TransactionState::SHRINKING) {
    throw TransactionAbortException(txn->get_transaction_id(),
                                    AbortReason::LOCK_ON_SHIRINKING);
  }
  if (txn_state == TransactionState::DEFAULT) {
    txn_state = TransactionState::GROWING;
  }
  return true;
}
```

**示例**：T1 已经释放过一把锁，状态进入 `SHRINKING` 后又想读取新记录，这时会抛出 `LOCK_ON_SHIRINKING` 异常。

**输入**：`txn` 是准备申请锁的事务对象。

**输出**：返回布尔值，表示是否允许继续加锁。

## 表级共享锁

**含义**：表级共享锁是锁住整张表的读锁。

**作用**：它允许多个事务同时读同一张表，但阻止其他事务对整张表做排他修改。

```cpp
// src/transaction/concurrency/lock_manager.cpp:877-892
bool LockManager::lock_shared_on_table(Transaction* txn, int tab_fd) {
  std::lock_guard lock(latch_);

  if (!check_lock(txn)) {
    return false;
  }

  LockDataId lock_data_id(tab_fd, LockDataType::TABLE);
  auto&& it = lock_table_.find(lock_data_id);
  if (it == lock_table_.end()) {
    it = lock_table_
             .emplace(std::piecewise_construct,
                      std::forward_as_tuple(lock_data_id),
                      std::forward_as_tuple())
             .first;
```

**输入**：`txn` 是申请锁的事务，`tab_fd` 是目标表文件描述符。

**输出**：返回是否成功获取表级 S 锁。

**锁说明**：这是事务级表锁。

**范围**：锁住 `tab_fd` 对应的整张表。

**类型**：共享读锁 S。

**生命周期**：扫描或读操作前申请，事务提交或回滚时统一释放。

## 表级排他锁

**含义**：表级排他锁是锁住整张表的写锁。

**作用**：它阻止其他事务同时读写这张表。

```cpp
// src/transaction/concurrency/lock_manager.cpp:980-1008
bool LockManager::lock_exclusive_on_table(Transaction* txn, int tab_fd) {
  std::lock_guard lock(latch_);

  if (!check_lock(txn)) {
    return false;
  }

  LockDataId lock_data_id(tab_fd, LockDataType::TABLE);

  auto it = lock_table_.find(lock_data_id);
  if (it == lock_table_.end()) {
    it = lock_table_
             .emplace(std::piecewise_construct,
                      std::forward_as_tuple(lock_data_id),
                      std::forward_as_tuple())
             .first;
  } else {
    auto& lock_request_queue = it->second;
    for (auto& lock_request : lock_request_queue.request_queue_) {
      if (lock_request.txn_id_ == txn->get_transaction_id()) {
        if (lock_request.lock_mode_ == LockMode::EXCLUSIVE) {
          return true;
        }
```

**输入**：`txn` 是申请锁的事务，`tab_fd` 是目标表文件描述符。

**输出**：返回是否成功获取表级 X 锁。

**锁说明**：这是事务级表锁。

**范围**：锁住 `tab_fd` 对应的整张表。

**类型**：排他写锁 X。

**生命周期**：写操作前申请，事务提交或回滚时统一释放。

## 记录级锁

**含义**：记录级锁是锁住单条记录的事务锁。

**作用**：它让不同事务可以并发访问同一张表中的不同记录，从而比整表加锁粒度更细。

```cpp
// src/transaction/concurrency/lock_manager.cpp:644-653
bool LockManager::lock_shared_on_record(Transaction* txn, const Rid& rid,
                                        int tab_fd) {
  std::lock_guard lock(latch_);

  if (!check_lock(txn)) {
    return false;
  }

  LockDataId lock_data_id(tab_fd, rid, LockDataType::RECORD);
  auto&& it = lock_table_.find(lock_data_id);
```

```cpp
// src/transaction/concurrency/lock_manager.cpp:745-753
bool LockManager::lock_exclusive_on_record(Transaction* txn, const Rid& rid,
                                           int tab_fd) {
  std::lock_guard lock(latch_);

  if (!check_lock(txn)) {
    return false;
  }

  LockDataId lock_data_id(tab_fd, rid, LockDataType::RECORD);
```

**示例**：T1 更新 page 3 slot 5 的记录时，只需要锁住这个 Rid 对应的记录，不必锁住整张表。

**锁说明**：这是事务级记录锁。

**范围**：锁住 `tab_fd + Rid` 唯一确定的单条记录。

**类型**：读操作申请 S 锁，写操作申请 X 锁。

**生命周期**：访问记录前申请，事务提交或回滚时统一释放。

## 意向锁

**含义**：意向锁是加在表上的提示锁，用来表示事务准备在表内部的记录上加锁。

**作用**：它让表级锁和记录级锁能够安全共存。

```cpp
// src/transaction/concurrency/lock_manager.cpp:1115-1123
bool LockManager::lock_IS_on_table(Transaction* txn, int tab_fd) {
  std::lock_guard lock(latch_);

  if (!check_lock(txn)) {
    return false;
  }

  LockDataId lock_data_id(tab_fd, LockDataType::TABLE);
  auto&& it = lock_table_.find(lock_data_id);
```

```cpp
// src/transaction/concurrency/lock_manager.cpp:1205-1219
bool LockManager::lock_IX_on_table(Transaction* txn, int tab_fd) {
  std::lock_guard lock(latch_);

  LockDataId lock_data_id(tab_fd, LockDataType::TABLE);
  auto&& it = lock_table_.find(lock_data_id);
  if (it == lock_table_.end()) {
    it = lock_table_
             .emplace(std::piecewise_construct,
                      std::forward_as_tuple(lock_data_id),
                      std::forward_as_tuple())
             .first;
```

**示例**：T1 想更新表内某一行时，可以先在表上加 IX 锁，再在具体记录上加 X 锁。

**锁说明**：这是事务级表锁。

**范围**：锁住 `tab_fd` 对应的整张表，但语义上表示即将访问表内更细粒度资源。

**类型**：IS 是意向共享锁，IX 是意向排他锁，SIX 是共享加意向排他组合锁。

**生命周期**：申请记录级锁前申请，事务提交或回滚时统一释放。

## 锁兼容关系

**含义**：锁兼容关系决定一个新锁请求是否能和已有锁同时存在。

**作用**：它是 LockManager 判断等待、授权或回滚的依据。

| 已有锁 \ 新请求 | IS | IX | S | SIX | X |
|----------------|----|----|---|-----|---|
| IS | 兼容 | 兼容 | 兼容 | 兼容 | 冲突 |
| IX | 兼容 | 兼容 | 冲突 | 冲突 | 冲突 |
| S | 兼容 | 冲突 | 兼容 | 冲突 | 冲突 |
| SIX | 兼容 | 冲突 | 冲突 | 冲突 | 冲突 |
| X | 冲突 | 冲突 | 冲突 | 冲突 | 冲突 |

**示例**：多个事务可以同时持有 IS 锁，因为它们只是声明会读表内记录；但如果已有 X 锁，任何其他锁都不能兼容。

## 死锁预防

**含义**：RMDB 参考实现使用 wait-die 思路预防死锁。

**背景**：当两个事务互相等待对方持有的锁时，就会形成死锁——双方都无法继续。wait-die 是一种基于事务年龄的死锁预防策略：每个事务有一个唯一的事务编号（由 TransactionManager 的 `next_txn_id_` 递增分配），编号越小的事务越老、优先级越高。

**规则**：老事务遇到冲突时可以等待年轻事务释放锁，年轻事务遇到老事务时只能直接中止（抛出 DEADLOCK_PREVENTION 异常）——因为"老等少"不会形成环，"少等老"则被禁止。

**作用**：通过禁止"年轻事务等待老事务"打破所有可能的环形等待，无需死锁检测就能保证无死锁。

```cpp
// src/transaction/concurrency/lock_manager.cpp:819-829
if (lock_request_queue.group_lock_mode_ != GroupLockMode::NON_LOCK) {
  if (txn->get_transaction_id() > lock_request_queue.oldest_txn_id_) {
    throw TransactionAbortException(txn->get_transaction_id(),
                                    AbortReason::DEADLOCK_PREVENTION);
  }

  lock_request_queue.oldest_txn_id_ = txn->get_transaction_id();
  lock_request_queue.request_queue_.emplace_back(txn->get_transaction_id(),
                                                 LockMode::EXCLUSIVE);
```

**示例**：T1 的事务编号小于 T2，表示 T1 更老；如果 T2 想拿 T1 正持有的冲突锁，T2 会被中止，避免双方互等。

## unlock

**含义**：`unlock` 释放事务在某个资源上的锁。

**作用**：它删除请求队列中属于该事务的请求，重算队列锁模式，并唤醒等待中的事务。

```cpp
// src/transaction/concurrency/lock_manager.cpp:1353-1365
bool LockManager::unlock(Transaction* txn, const LockDataId& lock_data_id) {
  std::lock_guard lock(latch_);

  auto& txn_state = txn->get_state();
  if (txn_state == TransactionState::COMMITTED ||
      txn_state == TransactionState::ABORTED) {
    return false;
  }

  if (txn_state == TransactionState::GROWING) {
    txn_state = TransactionState::SHRINKING;
  }
```

**输入**：`txn` 是释放锁的事务，`lock_data_id` 是要释放的锁资源。

**输出**：返回是否完成释放流程。

**锁说明**：`unlock` 释放事务级锁，同时用 `latch_` 保护 LockManager 内部锁表。

**范围**：事务级锁的释放范围是单个 `LockDataId`，内部 `latch_` 的保护范围是 `lock_table_` 和 `gap_lock_table_`。

**类型**：被释放的事务锁可能是 IS、IX、S、SIX 或 X，内部 `latch_` 是互斥锁。

**生命周期**：事务锁从申请成功后持有到 `unlock` 删除请求；内部 `latch_` 只在 `unlock` 函数执行期间持有。

上一节：[03-transaction-manager.md](./03-transaction-manager.md) | 下一节：[05-gap-lock.md](./05-gap-lock.md)
