# 事务管理器

## 基本职责

**含义**：`TransactionManager` 是事务层对外的门面类。

**作用**：它负责创建事务、提交事务、回滚事务、维护全局事务表，并把事务结束时的锁释放和写集清理串起来。

**场景**：rmdb 主循环和执行层在 SQL 开始、提交、异常回滚时调用它。

```cpp
// src/transaction/transaction_manager.h:23-45
class TransactionManager {
 public:
  explicit TransactionManager(
      LockManager* lock_manager, SmManager* sm_manager,
      ConcurrencyMode concurrency_mode = ConcurrencyMode::TWO_PHASE_LOCKING) {
    sm_manager_ = sm_manager;
    lock_manager_ = lock_manager;
    concurrency_mode_ = concurrency_mode;
  }

  Transaction* begin(Transaction* txn, LogManager* log_manager);

  void commit(Transaction* txn, LogManager* log_manager);

  void abort(Transaction* txn, LogManager* log_manager);
```

**示例**：执行 `BEGIN; UPDATE student SET age = 19 WHERE id = 1; COMMIT;` 时，TransactionManager 先创建事务，再在 COMMIT 时释放锁和清理写集。

## begin

**含义**：`begin` 开始一个事务。

**作用**：它在没有传入事务对象时创建新事务，分配开始时间戳，并把事务加入全局事务表。

```cpp
// src/transaction/transaction_manager.cpp:26-47
Transaction* TransactionManager::begin(Transaction* txn,
                                       LogManager* log_manager) {
  if (txn == nullptr) {
    txn = new Transaction(next_txn_id_++);
  }
  txn->set_start_ts(next_timestamp_++);
  latch_.lock();
  txn_map.emplace(txn->get_transaction_id(), txn);
  latch_.unlock();
#ifdef ENABLE_LOGGING
  auto* begin_log_record = new BeginLogRecord(txn->get_transaction_id());
  begin_log_record->prev_lsn_ = txn->get_prev_lsn();
  txn->set_prev_lsn(log_manager->add_log_to_buffer(begin_log_record));
  delete begin_log_record;
#endif
  return txn;
}
```

**输入**：`txn` 是可复用事务指针，空指针表示创建新事务；`log_manager` 用于写 BEGIN 日志。

**输出**：返回已经登记到 `txn_map` 的事务对象指针。

**示例**：隐式单条 SQL 通常会传入空指针，让系统自动创建一个事务。

**锁说明**：这里使用的是 `latch_` 这个 C++ mutex，不是事务级锁。

**范围**：`latch_` 只保护 `txn_map` 这个全局事务表。

**类型**：`latch_` 是互斥锁，等价于排他保护。

**生命周期**：进入插入 `txn_map` 的临界区前加锁，插入完成后立即释放。

## commit

**含义**：`commit` 提交一个事务。

**作用**：它删除写集中的撤销记录，释放事务持有的所有锁，写 COMMIT 日志，并把事务状态改成 `COMMITTED`。

```cpp
// src/transaction/transaction_manager.cpp:55-84
void TransactionManager::commit(Transaction* txn, LogManager* log_manager) {
  for (auto& it : *txn->get_write_set()) {
    delete it;
  }

  auto&& lock_set = txn->get_lock_set();
  for (auto& it : *lock_set) {
    lock_manager_->unlock(txn, it);
  }
  lock_set->clear();
#ifdef ENABLE_LOGGING
  auto* commit_log_record = new CommitLogRecord(txn->get_transaction_id());
  commit_log_record->prev_lsn_ = txn->get_prev_lsn();
  txn->set_prev_lsn(log_manager->add_log_to_buffer(commit_log_record));
  delete commit_log_record;
#endif
  txn->set_state(TransactionState::COMMITTED);
}
```

**输入**：`txn` 是要提交的事务，`log_manager` 用于写 COMMIT 日志。

**输出**：没有返回值，副作用是事务完成、锁释放、状态更新。

**示例**：如果 T1 修改了三条记录，commit 不需要再改数据，因为数据已经在执行 UPDATE 时写入；commit 只需要确认事务结束并清理事务资源。

**锁说明**：commit 释放的是事务级锁。

**范围**：释放范围是 `lock_set_` 中记录的所有表锁、记录锁和间隙锁。

**类型**：释放的锁可能是 IS、IX、S、SIX 或 X。

**生命周期**：这些锁从执行层申请开始持有，到 commit 调用 `unlock` 时结束。

## abort

**含义**：`abort` 回滚一个事务。

**作用**：它逆序遍历写集，逐条撤销 INSERT、DELETE 和 UPDATE，然后释放所有锁并把状态改成 `ABORTED`。

```cpp
// src/transaction/transaction_manager.cpp:91-113
void TransactionManager::abort(Transaction* txn, LogManager* log_manager) {
  std::lock_guard lock(latch_);

  auto&& write_set = txn->get_write_set();
  auto* context = new Context(lock_manager_, log_manager, txn);
  for (auto&& it = write_set->rbegin(); it != write_set->rend(); ++it) {
    auto& write_record = *it;
    auto& table_name = write_record->GetTableName();
    auto& table_meta = sm_manager_->db_.get_table(table_name);
    auto& fh = sm_manager_->fhs_[table_name];
    switch (write_record->GetWriteType()) {
      case WType::INSERT_TUPLE: {
        auto& rid = write_record->GetRid();
        auto& record = write_record->GetRecord();
        fh->delete_record(rid, context);
```

**输入**：`txn` 是要回滚的事务，`log_manager` 用于写撤销过程中的日志和 ABORT 日志。

**输出**：没有返回值，副作用是撤销数据修改、释放锁、状态更新。

**示例**：如果事务先 INSERT 后 UPDATE，回滚时必须先撤销 UPDATE，再撤销 INSERT，因为后一步可能依赖前一步产生的数据。

## 写集回滚规则

**含义**：写集回滚是把已经执行过的物理修改反向执行。

**作用**：它保证事务的原子性，也就是事务中的修改要么全部保留，要么全部撤销。

| 写操作 | 回滚动作 | 索引维护 |
|--------|----------|----------|
| INSERT | 删除刚插入的记录 | 删除对应索引项 |
| DELETE | 重新插入被删记录 | 重新插入对应索引项 |
| UPDATE | 把新记录恢复成旧记录 | 如果改到索引列，删除新键并插入旧键 |

**示例**：如果事务把学生 id 从 1 改成 2，并且 id 上有索引，那么回滚要删除键 2 对应的索引项，再插入键 1 对应的索引项。

**术语**：这里的键指索引列拼接出的值，索引项是键和 Rid 组成的键值对。

## 索引回滚中的锁

**级别**：这里使用的是索引对象内部的 Page 级读写锁包装接口 `rw_latch_`，不是事务级锁表中的锁。

**范围**：`ih->rw_latch_.WLock()` 保护的是整个索引句柄对应的 B+ 树操作入口。

**类型**：`WLock` 是写锁，表示排他修改索引结构。

**生命周期**：回滚每个索引项前加写锁，调用 `delete_entry` 或 `insert_entry` 完成后立即 `WUnlock`。

## 全局事务表

**含义**：`txn_map` 是事务编号到事务对象指针的全局映射。

**作用**：它让系统可以根据事务编号找到仍在运行的事务对象。

```cpp
// src/transaction/transaction_manager.h:60-83
Transaction* get_transaction(txn_id_t txn_id) {
  if (txn_id == INVALID_TXN_ID) {
    return nullptr;
  }

  std::unique_lock lock(latch_);
  if (txn_map.find(txn_id) == txn_map.end()) {
    return nullptr;
  }

  auto* res = txn_map[txn_id];
  if (res->get_state() == TransactionState::COMMITTED ||
      res->get_state() == TransactionState::ABORTED) {
    delete res;
    txn_map.erase(txn_id);
    return nullptr;
  }
```

**示例**：如果查询一个已经提交的事务，`get_transaction` 会删除事务对象并从 `txn_map` 中移除它。

上一节：[02-transaction-data-structures.md](./02-transaction-data-structures.md) | 下一节：[04-lock-manager.md](./04-lock-manager.md)
