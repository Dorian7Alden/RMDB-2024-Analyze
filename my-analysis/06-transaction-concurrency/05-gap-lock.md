# 间隙锁

## 为什么需要间隙锁

**含义**：间隙锁锁住的是索引 key 之间的范围，而不仅是某条已有记录。

**作用**：没有间隙锁时，可串行化隔离级别会出现幻读问题——事务 T1 第一次读取条件 `age > 18` 得到 3 行，T2 在此期间插入了一条 `age = 25` 的新记录并提交，T1 再读同一条件就得到了 4 行。

**场景**：间隙锁由 SeqScanExecutor、IndexScanExecutor、InsertExecutor、DeleteExecutor 和 UpdateExecutor 在可串行化隔离下调用。

**术语**：这里的索引 key 指索引字段的值，不含 Rid。

## Gap

**含义**：`Gap` 是一个索引范围，由若干列上的左右条件组成。

**作用**：它用索引条件精确描述“锁住哪些 key 之间的空间”，从而阻止并发插入。

**示例**：`WHERE age > 18 AND age < 30 AND score >= 60` 会生成一个两列的 Gap，其中第一列有左闭 18 和右开 30，第二列有左闭 60 无右界。

```cpp
// src/transaction/txn_defs.h:105-109
class Gap {
 public:
  Gap() = default;

  explicit Gap(std::vector<std::pair<CondOp, CondOp> >& index_conds) {
    index_conds_ = index_conds;
  }
```

**输入**：构造函数输入 `index_conds`，每个元素是该列的一个左右条件对。

**输出**：构造出一个表示范围的对象。

## 间隙相交判断

**含义**：`isCoincide` 判断两个 Gap 的范围是否有交集。

**作用**：间隙锁的冲突检查依赖它，因为排他间隙锁和任何与其范围有交集的锁都不能共存。

```cpp
// src/transaction/txn_defs.h:117-120
bool isCoincide(const Gap& gap) const {
  for (std::size_t i = 0; i < index_conds_.size(); ++i) {
    auto& lhs_cond = index_conds_[i].first;
    auto& rhs_cond = gap.index_conds_[i].second;
    if (lhs_cond.op != OP_INVALID && rhs_cond.op != OP_INVALID) {
```

**示例**：A 的 Gap 是 `age in [18, 30)`，B 的 Gap 是 `age in [25, 40)`，二者有交集 `[25, 30)`，所以冲突。

## lock_shared_on_gap

**含义**：在某个索引间隙上申请共享锁。

**作用**：读操作产生的索引范围扫描需要 S 间隙锁，以防止扫描过程中其他事务在范围中点插入新记录。

```cpp
// src/transaction/concurrency/lock_manager.cpp:42-61
bool LockManager::lock_shared_on_gap(Transaction* txn, IndexMeta& index_meta,
                                     Gap& gap, int tab_fd) {
  std::lock_guard lock(latch_);

  if (!check_lock(txn)) {
    return false;
  }

  LockDataId lock_data_id(tab_fd, index_meta, gap, LockDataType::GAP);
  auto&& it = gap_lock_table_.find(index_meta);
  if (it == gap_lock_table_.end()) {
    it =
        gap_lock_table_
            .emplace(std::piecewise_construct,
                     std::forward_as_tuple(index_meta), std::forward_as_tuple())
            .first;
```

**输入**：`txn` 是申请锁的事务，`index_meta` 是索引元数据，`gap` 是索引范围，`tab_fd` 是表文件描述符。

**输出**：返回是否成功获取 S 间隙锁。

**锁说明**：这是事务级间隙锁。

**范围**：锁住 `index_meta` 对应索引上由 `gap` 描述的范围空间。

**类型**：共享锁 S，多个读事务可以共享同一间隙。

**生命周期**：读取索引范围前申请，事务提交或回滚时统一释放。

## lock_exclusive_on_gap

**含义**：在某个索引间隙上申请排他锁。

**作用**：INSERT、DELETE 和 UPDATE 操作需要 X 间隙锁，确保没有其他事务正在读或写这个范围。

```cpp
// src/transaction/concurrency/lock_manager.cpp:232-249
bool LockManager::lock_exclusive_on_gap(Transaction* txn, IndexMeta& index_meta,
                                        Gap& gap, int tab_fd) {
  std::lock_guard lock(latch_);

  if (!check_lock(txn)) {
    return false;
  }

  LockDataId lock_data_id(tab_fd, index_meta, gap, LockDataType::GAP);
  auto it = gap_lock_table_.find(index_meta);
  if (it == gap_lock_table_.end()) {
    it =
        gap_lock_table_
            .emplace(std::piecewise_construct,
                     std::forward_as_tuple(index_meta), std::forward_as_tuple())
            .first;
```

**输入**：`txn` 是申请锁的事务，`index_meta` 是索引元数据，`gap` 是索引范围，`tab_fd` 是表文件描述符。

**输出**：返回是否成功获取 X 间隙锁。

**锁说明**：这是事务级间隙锁。

**范围**：锁住 `index_meta` 对应索引上由 `gap` 描述的范围空间。

**类型**：排他锁 X，排他间隙锁与任何冲突范围的锁都不兼容。

**生命周期**：执行 INSERT、DELETE 或 UPDATE 修改索引键前申请，事务提交或回滚时统一释放。

## isSafeInGap

**含义**：判断一条记录是否可以被安全地放入某个间隙。

**作用**：InsertExecutor 在插入新记录时调用它，检查目标索引范围是否存在其他事务持有的冲突锁。

```cpp
// src/transaction/concurrency/lock_manager.cpp:498-504
bool LockManager::isSafeInGap(Transaction* txn, IndexMeta& index_meta,
                              RmRecord& record, int tab_fd) {
  std::unique_lock lock(latch_);

  auto predicate_manager = PredicateManager(index_meta);

  // 手动写个 cond index_col = val
  std::vector<Condition> conds(index_meta.cols.size());
```

**输入**：`txn` 是插入事务，`index_meta` 是索引元数据，`record` 是待插入的记录，`tab_fd` 是表文件描述符。

**输出**：返回是否可以安全插入，`false` 表示间隙被其他事务锁住。

**示例**：事务 T2 尝试插入 `age = 25` 的记录，但 T1 已经对 `[20, 30)` 范围持有 S 间隙锁，那么 T2 的安全检查失败。

**锁说明**：这是事务级间隙锁的安全检查。

**范围**：检查 `index_meta` 对应索引上是否已有事务锁住了 `record` 落入的间隙范围。

**类型**：扫描已存在的间隙锁，检查是否有其他事务的冲突锁。

**生命周期**：检查操作不持有新锁，只在 `lock_` 临界区内读取 `gap_lock_table_` 完成。

## gap_lock_table 结构

**含义**：`gap_lock_table_` 是专门管理间隙锁的全局哈希表。

**作用**：它按 IndexMeta 分组，每个索引下用 LockDataId 映射到各自的请求队列，从而支持同一张表上不同索引的独立加锁。

```cpp
// src/transaction/concurrency/lock_manager.h:93-98
std::mutex latch_;
std::unordered_map<LockDataId, LockRequestQueue> lock_table_;
std::unordered_map<IndexMeta,
                   std::unordered_map<LockDataId, LockRequestQueue> >
    gap_lock_table_;
```

**示例**：student 表上有 `id` 索引和 `age` 索引，插入新记录时可能只在 `age` 索引上加间隙锁，`id` 索引不受影响。

上一节：[04-lock-manager.md](./04-lock-manager.md) | 下一节：[06-transaction-interaction.md](./06-transaction-interaction.md)
