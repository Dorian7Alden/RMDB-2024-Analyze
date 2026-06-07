# 日志管理器

## 基本职责

**含义**：`LogManager` 是所有日志写入的统一入口。

**作用**：它负责给每条日志分配全局唯一的 LSN（Log Sequence Number，日志序列号）、把日志序列化到内存缓冲区、把缓冲区刷到磁盘。

**场景**：被 TransactionManager 在 begin/commit/abort 时调用，被执行器在 INSERT/DELETE/UPDATE 时调用。

```cpp
// src/recovery/log_manager.h:436-489
class LogManager {
 public:
  explicit LogManager(DiskManager* disk_manager)
      : disk_manager_(disk_manager),
        run_background_thread_(true),
        log_flush_interval_(std::chrono::seconds(1)) {
    background_thread_ = std::thread([this] { background_flush(); });
  }

  lsn_t add_log_to_buffer(LogRecord* log_record);
  void flush_log_to_disk();

  inline lsn_t get_persist_lsn() const { return persist_lsn_; }
  inline void set_global_lsn(lsn_t global_lsn) {
    global_lsn_.store(global_lsn);
  }

 private:
  void background_flush();
  std::atomic<lsn_t> global_lsn_{0};
  std::mutex latch_;
  LogBuffer log_buffer_;
  lsn_t persist_lsn_{};
  DiskManager* disk_manager_;
};
```

**示例**：当 TransactionManager::commit 写 COMMIT 日志时，它调用 `log_manager->add_log_to_buffer(commit_log_record)`，LogManager 分配 LSN、序列化并追加到缓冲区。

## add_log_to_buffer

**含义**：把一条日志记录序列化后追加到日志缓冲区。

**作用**：如果缓冲区满了，先刷盘再追加，保证不会溢出。

```cpp
// src/recovery/log_manager.cpp:19-29
lsn_t LogManager::add_log_to_buffer(LogRecord* log_record) {
  latch_.lock();
  if (log_buffer_.is_full(log_record->log_tot_len_)) {
    flush_log_to_disk();
  }
  log_record->lsn_ = global_lsn_++;
  log_record->serialize(log_buffer_.buffer_ + log_buffer_.offset_);
  log_buffer_.offset_ += log_record->log_tot_len_;
  latch_.unlock();
  return log_record->lsn_;
}
```

**输入**：`log_record` 是已经填充好日志头和日志体的日志对象指针。

**输出**：返回分配给的 LSN，调用者把这 LSN 设置到事务的 `prev_lsn_` 上。

**示例**：执行三条 INSERT 后，缓冲区里的 offset 递增了三次 `log_tot_len_`，每条日志分配的 LSN 依次是 0, 1, 2。

**锁说明**：使用的是 `latch_` 这个 C++ mutex。

**范围**：保护 `log_buffer_` 的完整性和 `global_lsn_` 的递增。

**类型**：互斥锁，一次只允许一个线程追加日志。

**生命周期**：进入 `add_log_to_buffer` 时加锁，函数结束时释放。

## flush_log_to_disk

**含义**：把日志缓冲区中的全部内容一次性写入磁盘。

**作用**：它调用 `disk_manager_->write_log` 执行实际的磁盘写入，然后把缓冲区清空并更新 `persist_lsn_`。

```cpp
// src/recovery/log_manager.cpp:35-43
void LogManager::flush_log_to_disk() {
  if (log_buffer_.offset_ == 0) {
    return;
  }
  disk_manager_->write_log(log_buffer_.buffer_, log_buffer_.offset_);
  log_buffer_.offset_ = 0;
  persist_lsn_ = global_lsn_ - 1;
}
```

**输入**：无参数，操作对象是内部的 `log_buffer_`。

**输出**：无返回值，副作用是缓冲区内容已持久化到磁盘日志文件。

**示例**：如果 `global_lsn_` 当前是 5（表示已分配了 0 到 4），刷盘后 `persist_lsn_` 变为 4，表示 LSN 0 到 4 的日志都已安全落盘。

## 后台刷盘线程

**含义**：LogManager 在构造时启动一个后台线程，定时自动调用 `flush_log_to_disk`。

**作用**：它保证日志不会长时间留在内存中，即使没有单个事务达到缓冲区满的条件。

```cpp
// src/recovery/log_manager.h:468-477
void background_flush() {
  std::unique_lock lk(latch_);
  while (run_background_thread_) {
    if (cv_.wait_for(lk, log_flush_interval_,
                     [this] { return !run_background_thread_; })) {
      break;
    }
    flush_log_to_disk();
  }
}
```

**含义**：线程每 1 秒醒来一次，把缓冲区中的日志刷到磁盘，除非 `run_background_thread_` 被设为 false。

**作用**：即使事务不频繁，日志也不会在内存中积压太久。

**锁说明**：后台线程使用 `latch_` 和 `cv_` 控制刷盘时机。

**范围**：`latch_` 保护 `run_background_thread_` 开关，`flush_log_to_disk` 内部会再次争用 `latch_` 来保护缓冲区。

**类型**：`latch_` 是互斥锁，`cv_` 是条件变量。

**生命周期**：线程在 LogManager 构造时创建，析构时通过 `run_background_thread_ = false` 和 `cv_.notify_one()` 安全退出。

## global_lsn_ 与 persist_lsn_

**含义**：`global_lsn_` 是已分配的最大 LSN 加 1，`persist_lsn_` 是已持久化的最大 LSN。

**作用**：这两个计数器一起定义日志的"安全窗口"——`global_lsn_ - 1` 到 `persist_lsn_ + 1` 之间的日志在内存但不是磁盘。

**示例**：如果 `global_lsn_ = 10`、`persist_lsn_ = 7`，说明 LSN 0 到 7 的日志已安全落盘，LSN 8 和 9 还在缓冲区中，系统崩溃时 8 和 9 会丢失。

## 恢复时的 LSN 恢复

**含义**：系统重启恢复后，`global_lsn_` 和 `persist_lsn_` 必须恢复到崩溃前的状态。

**作用**：如果不恢复，新分配的 LSN 会和已存在日志的 LSN 重复，导致恢复逻辑错乱。

```cpp
// src/recovery/log_recovery.cpp:202-203
log_manager_->set_global_lsn(max_lsn + 1);
log_manager_->set_persist_lsn(max_lsn);
```

**含义**：`max_lsn` 是 analyze 阶段找到的最大 LSN，`global_lsn_` 设为 `max_lsn + 1` 保证新日志的 LSN 不会重复。

上一节：[02-recovery-data-structures.md](./02-recovery-data-structures.md) | 下一节：[04-recovery-manager.md](./04-recovery-manager.md)
