# 恢复管理器

## 基本职责

**含义**：`RecoveryManager` 是崩溃恢复算法的实现者。

**作用**：它实现接近 ARIES 的三步恢复算法——analyze（分析）、redo（重做）、undo（撤销），把数据库恢复到崩溃前的一致性状态。

**场景**：系统启动时，如果检测到上次是非正常退出（日志文件存在且没有检查点标记），就创建 RecoveryManager 并依次调用三个方法。

```cpp
// src/recovery/log_recovery.h:27-48
class RecoveryManager {
 public:
  RecoveryManager(DiskManager* disk_manager,
                  BufferPoolManager* buffer_pool_manager, SmManager* sm_manager,
                  LogManager* log_manager,
                  TransactionManager* transaction_manager)
      : disk_manager_(disk_manager),
        buffer_pool_manager_(buffer_pool_manager),
        sm_manager_(sm_manager),
        log_manager_(log_manager),
        transaction_manager_(transaction_manager),
        transaction_(666) {}

  // 666 是一个虚拟事务编号，仅用于恢复期间调用记录层和索引层接口
  // 恢复时原来的事务对象已销毁，需要用这个占位事务来推动 redo/undo 操作

  void analyze();
  void redo();
  void undo();

 private:
  std::unordered_map<txn_id_t, lsn_t> active_txn_;
  std::unordered_map<lsn_t, int> lsn_mapping_;
  std::deque<lsn_t> dirty_page_table_;
```

**含义**：三个关键数据结构分别维护活跃事务、日志定位和脏页信息。

| 数据成员 | 含义 | 由哪个阶段填充 |
|----------|------|--------------|
| `active_txn_` | 映射：事务编号 → 该事务最后一条日志的 LSN | analyze |
| `lsn_mapping_` | 映射：LSN → 该日志在文件中的偏移量 | analyze |
| `dirty_page_table_` | 需要 redo 的日志 LSN 列表 | analyze |

## 阶段 1：analyze

**含义**：analyze 阶段扫描整个日志文件，分类每条日志，输出活跃事务、脏页表和 LSN 映射。

```cpp
// src/recovery/log_recovery.cpp:19-205
void RecoveryManager::analyze() {
  lsn_t max_lsn = INVALID_LSN;
  txn_id_t max_txn_id = INVALID_TXN_ID;
  std::size_t log_offset = 0;
  size_t read_bytes;

  while ((read_bytes = disk_manager_->read_log(buffer_.buffer_, LOG_BUFFER_SIZE,
                                               log_offset)) > 0) {
    while (buffer_.offset_ + LOG_HEADER_SIZE <= read_bytes) {
      auto& log_size = *reinterpret_cast<const uint32_t*>(
          buffer_.buffer_ + buffer_.offset_ + OFFSET_LOG_TOT_LEN);
      if (log_size > read_bytes - buffer_.offset_) {
        break;
      }
      auto& log_type = *reinterpret_cast<const LogType*>(
          buffer_.buffer_ + buffer_.offset_ + OFFSET_LOG_TYPE);
      switch (log_type) {
        case BEGIN: {
          auto* log = new BeginLogRecord;
          log->deserialize(buffer_.buffer_ + buffer_.offset_);
          active_txn_.emplace(log->log_tid_, log->lsn_);
          lsn_mapping_.emplace(log->lsn_, log_offset + buffer_.offset_);
```

**输入**：磁盘上的日志文件 + LogBuffer 读入缓冲区。

**输出**：`active_txn_`、`lsn_mapping_`、`dirty_page_table_` 三个内部数据结构。

**含义**：analyze 对每条日志的处理规则如下。

| 日志类型 | 对 active_txn_ | 对 dirty_page_table_ | 说明 |
|----------|---------------|---------------------|------|
| BEGIN | 插入该事务 | 不处理 | 新事务开始活跃 |
| COMMIT | 删除该事务 | 不处理 | 事务已提交，无需 undo |
| ABORT | 删除该事务 | 不处理 | 事务已回滚，无需 undo |
| INSERT | 更新该事务的 LSN | 如果页 LSN 更旧则加入 | 数据可能未落盘 |
| DELETE | 更新该事务的 LSN | 如果页 LSN 更旧则加入 | 数据可能未落盘 |
| UPDATE | 更新该事务的 LSN | 如果页 LSN 更旧则加入 | 数据可能未落盘 |

**示例**：T5 有两条日志 INSERT lsn=8 和 UPDATE lsn=12。analyze 结束后 `active_txn_[5] = 12`，`lsn_mapping_[8] = 200`，`lsn_mapping_[12] = 360`。

**脏页判断**：如果日志涉及的数据页上记录的 LSN 小于本条日志的 LSN，说明这条日志的操作没有成功落盘到数据页，需要 redo。同一页可能在 `dirty_page_table_` 中出现多次（每条需要重做的日志操作都会记录一个 entry），这不是重复——redo 需要按顺序重放每条操作。

## 阶段 2：redo

**含义**：redo 阶段按 dirty_page_table_ 的顺序，从早到晚逐条重新执行操作。

**作用**：它确保已提交事务的修改被真正写入数据页和索引。

```cpp
// src/recovery/log_recovery.cpp:210-298
void RecoveryManager::redo() {
  for (auto& lsn : dirty_page_table_) {
    int log_offset = lsn_mapping_[lsn];
    disk_manager_->read_log(buffer_.buffer_, LOG_BUFFER_SIZE, log_offset);
    auto& log_type =
        *reinterpret_cast<const LogType*>(buffer_.buffer_ + OFFSET_LOG_TYPE);
    switch (log_type) {
      case INSERT: {
        auto log = new InsertLogRecord;
        log->deserialize(buffer_.buffer_);
        auto fh = sm_manager_->fhs_.at(log->table_name_).get();
        fh->insert_record(log->rid_, log->insert_value_.data);
        // redo 索引
        auto& indexes = sm_manager_->db_.get_table(log->table_name_).indexes;
        for (auto& [index_name, index_meta] : indexes) {
          char* key = new char[index_meta.col_tot_len];
          auto& ih = sm_manager_->ihs_.at(index_name);
          for (auto& [offset, col_meta] : index_meta.cols) {
            memcpy(key + offset, log->insert_value_.data + col_meta.offset,
                   col_meta.len);
          }
          ih->insert_entry(key, log->rid_, &transaction_);
          delete[] key;
        }
```

**输入**：`dirty_page_table_` 和 `lsn_mapping_` 决定重做范围。

**输出**：所有需要重做操作的数据和索引已更新到最新状态。

**含义**：每种操作的重做规则。

| 操作 | redo 动作 | 索引 redo |
|------|----------|----------|
| INSERT | 重新调用 `insert_record` | 重新调用 `insert_entry` |
| DELETE | 重新调用 `delete_record` | 重新调用 `delete_entry` |
| UPDATE | 重新调用 `update_record` 写入新值 | 删除旧键并插入新键 |
| BEGIN/COMMIT/ABORT | 跳过 | 不需要 |

**示例**：某条 INSERT 日志在崩溃前写入了日志但数据页还没刷盘，redo 时把它插入到对应记录页和索引页。

## 阶段 3：undo

**含义**：undo 阶段逆序撤销所有活跃事务的修改。

**作用**：它保证未提交事务的所有修改被彻底清除，数据库只保留已提交事务的结果。

```cpp
// src/recovery/log_recovery.cpp:305-437
void RecoveryManager::undo() {
  std::priority_queue<lsn_t> lsn_heap;
  for (auto& [_, lsn] : active_txn_) {
    lsn_heap.emplace(lsn);
  }

  lsn_t lsn;
  while (!lsn_heap.empty()) {
    lsn = lsn_heap.top();
    lsn_heap.pop();
    int log_offset = lsn_mapping_[lsn];
    disk_manager_->read_log(buffer_.buffer_, LOG_BUFFER_SIZE, log_offset);
    auto& log_type =
        *reinterpret_cast<const LogType*>(buffer_.buffer_ + OFFSET_LOG_TYPE);
    switch (log_type) {
      case INSERT: {
        auto log = new InsertLogRecord;
        log->deserialize(buffer_.buffer_);
        auto fh = sm_manager_->fhs_.at(log->table_name_).get();
        fh->delete_record(log->rid_, nullptr);
        // undo 索引
        auto& indexes = sm_manager_->db_.get_table(log->table_name_).indexes;
        for (auto& [index_name, index_meta] : indexes) {
          char* key = new char[index_meta.col_tot_len];
          auto& ih = sm_manager_->ihs_.at(index_name);
          for (auto& [offset, col_meta] : index_meta.cols) {
            memcpy(key + offset, log->insert_value_.data + col_meta.offset,
                   col_meta.len);
          }
          ih->delete_entry(key, &transaction_);
          delete[] key;
        }
        lsn = log->prev_lsn_;
        delete log;
        break;
      }
```

**输入**：`active_txn_` 和 `lsn_mapping_` 决定撤销范围。

**输出**：所有活跃事务的修改被反转，数据和索引一致。

**含义**：undo 使用大根堆来保证跨事务的逆序撤销——每次从所有活跃事务中取出 LSN 最大的一条日志执行反向操作。

| 操作 | undo 动作 | 索引 undo |
|------|----------|----------|
| INSERT | 删除刚插入的记录 | 删除对应索引项 |
| DELETE | 重新插入被删记录 | 重新插入对应索引项 |
| UPDATE | 恢复旧记录 | 删除新键并插入旧键 |
| BEGIN | 标记该事务链结束，不再继续 prev_lsn | 不需要 |

**示例**：T3 有 LSN=5 BEGIN、LSN=8 INSERT、LSN=12 UPDATE，undo 时先处理 LSN=12 UPDATE（恢复旧值），查到 prev_lsn=8 把 LSN=8 加入堆，再处理 LSN=8 INSERT（删除记录），查到 prev_lsn=5 加入堆，最后处理 LSN=5 BEGIN 停止。

**术语**：这里的索引键指从记录中按索引列取出的值，索引项是键与 Rid 组成的键值对。

## redo_indexes

**含义**：重新构建所有表的索引。

**作用**：当 redo 和 undo 过程中索引维护出现不一致时，提供最兜底的修复手段——全量重建索引。

```cpp
// src/recovery/log_recovery.cpp:442-459
void RecoveryManager::redo_indexes() {
  std::vector<std::string> col_names;
  auto* context = new Context(nullptr, nullptr, &transaction_);
  for (auto& [table_name, _] : sm_manager_->fhs_) {
    auto& table_meta = sm_manager_->db_.get_table(table_name);
    for (auto& [index_name, index_meta] : table_meta.indexes) {
      for (auto& [_, col] : index_meta.cols) {
        col_names.emplace_back(col.name);
      }
      sm_manager_->redo_index(table_name, table_meta, col_names, index_name,
                              context);
      col_names.clear();
    }
  }
  delete context;
}
```

**场景**：当前实现中 `redo_indexes` 被注释掉了，但在索引层有独立的重建逻辑可供调用。

上一节：[03-log-manager.md](./03-log-manager.md) | 下一节：[05-recovery-interaction.md](./05-recovery-interaction.md)
