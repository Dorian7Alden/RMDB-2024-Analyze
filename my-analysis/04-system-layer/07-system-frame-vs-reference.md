# 07. 框架与参考实现对比

框架（`db2026-x/src/system/`）提供了 SM 层的骨架，参赛者需要实现其中的关键方法并优化数据结构。

## 总体概览

| 组件 | 框架状态 | 需要做什么 |
|------|---------|-----------|
| `is_dir` | 已完整实现 | 无需修改 |
| `create_db` | 已完整实现 | 无需修改 |
| `drop_db` | 已完整实现 | 无需修改 |
| `show_tables` | 已完整实现 | 无需修改 |
| `desc_table` | 基本完整 | 有小差异（`index` 字段） |
| `open_db` | **空方法体** | 实现完整的数据库打开流程 |
| `close_db` | **空方法体** | 实现完整的数据库关闭流程 |
| `flush_meta` | **有 bug** | 添加 `std::ios::trunc` |
| `create_table` | **不完整** | 添加 `cols_map` 构建 |
| `drop_table` | **空方法体** | 实现级联删除流程 |
| `create_index` | **空方法体** | 实现全表扫描 + B+ 树构建 |
| `drop_index` | **空方法体** | 实现索引文件清理 |
| `show_indexs` | **不存在** | 添加索引列表查询 |
| `redo_index` | **不存在** | 添加恢复用索引重建 |
| 元数据结构 | 功能简化 | 添加 hash map 优化查找 |

## 逐项详析

### 1. open_db() — 空 → 完整

**框架**（`db2026-x/src/system/sm_manager.cpp:87-89`）：

```cpp
// db2026-x/src/system/sm_manager.cpp:87-89
void SmManager::open_db(const std::string& db_name) {
    // 空
}
```

**参考实现**（`src/system/sm_manager.cpp:90-120`）：

```cpp
// src/system/sm_manager.cpp:90-120
void SmManager::open_db(const std::string& db_name) {
  if (!is_dir(db_name)) throw DatabaseNotFoundError(db_name);
  if (!db_.name_.empty()) throw DatabaseExistsError(db_name);
  chdir(db_name.c_str());
  std::ifstream ifs(DB_META_NAME);
  ifs >> db_;  // 反序列化元数据
  for (auto& [table_name, tab_meta] : db_.tabs_) {
    fhs_[table_name] = rm_manager_->open_file(table_name);
    for (auto& [index_name, _] : tab_meta.indexes) {
      ihs_[index_name] = ix_manager_->open_index(index_name);
    }
  }
}
```

**为什么需要这些代码**：打开数据库不只是读元数据——还需要把数据库中的所有表文件和索引文件全部打开。这是**懒加载 vs 预加载**的选择。参考实现选择预加载，把所有句柄缓存好，后续操作直接从 `fhs_` / `ihs_` 取，不再需要打开文件。

### 2. close_db() — 空 → 完整

**框架**：空方法体。

**参考实现**（`src/system/sm_manager.cpp:134-163`）：约 30 行，包含 flush_meta → 清空 db_ → 关闭所有记录文件 → 关闭所有索引文件 → 清空 fhs_/ihs_ → chdir 返回。

**为什么需要这些代码**：关闭不是简单的"不管了"——必须先落盘元数据，再关闭所有文件句柄。如果先关闭文件再写元数据，缓冲池中的脏页可能还没写出，导致数据丢失。

### 3. flush_meta() — 缺少 trunc → 正确

**框架**（`db2026-x/src/system/sm_manager.cpp:94-98`）：

```cpp
// db2026-x/src/system/sm_manager.cpp:94-98
void SmManager::flush_meta() {
    std::ofstream ofs(DB_META_NAME);  // 没有 std::ios::trunc
    ofs << db_;
}
```

**参考实现**（`src/system/sm_manager.cpp:125-129`）：

```cpp
// src/system/sm_manager.cpp:125-129
void SmManager::flush_meta() {
  std::ofstream ofs(DB_META_NAME, std::ios::trunc);
  ofs << db_;
}
```

**为什么需要 `trunc`**：没有 `trunc` 时，如果新的元数据文本比旧文本短，旧文本的尾部会残留在文件中。

举例说明：

```
旧元数据（100 字节）
新元数据（80 字节，因为删了一张表）

没有 trunc：文件前 80 字节是新数据，后 20 字节是旧数据残留
            文件大小 = 100 字节

有 trunc：   文件 = 80 字节新数据
            文件大小 = 80 字节
```

虽然 `operator>>` 在反序列化时只读需要的部分（读到文件末尾自然停止），残留数据不会被读到——但反复 flush 会让文件只增不减。加 `trunc` 后每次都是全新写入，文件始终保持正确大小。

### 4. create_table() — 缺 cols_map → 完整

**框架**：构建了 `TabMeta`，但没有构建 `cols_map`。

**参考实现**：在字段列表构建完成后，添加了：

```cpp
// src/system/sm_manager.cpp:267-269
for (size_t i = 0; i < tab.cols.size(); ++i) {
  tab.cols_map.emplace(tab.cols[i].name, i);
}
```

**为什么需要 `cols_map`**：`TabMeta::get_col(col_name)` 在 Analyze 阶段被高频调用——每个列引用查一次。没有 `cols_map` 时每次 O(n) 遍历 `cols`，有 `cols_map` 时 O(1) 哈希查找。

对于包含几十列的表，每次 SQL 查询可能涉及 3-5 个列引用，O(n²) 的累积开销在反复查询中变得可感知。

### 5. drop_table() — 空 → 级联清理

**框架**：空方法体。

**参考实现**（`src/system/sm_manager.cpp:293-322`）：关闭并删除记录文件 → 遍历关闭并删除所有索引文件 → 清理 fhs_/ihs_ → 从 db_.tabs_ 移除 → flush_meta。

**为什么是级联操作**：删表不仅仅是删 .db 文件——表上可能有多个索引，每个索引的 .idx 文件也需要删除。漏删索引文件会导致磁盘上留下孤立的 .idx 文件（它们引用的表已经不存在了）。

### 6. create_index() — 空 → 65 行核心算法

**框架**：空方法体。

**参考实现**（`src/system/sm_manager.cpp:330-395`）：完整实现见 [05b-create-index-detail.md](./05b-create-index-detail.md)。

**为什么复杂**：建索引是整个 SM 层最复杂的操作——它需要遍历全表、构建键、维护 B+ 树、处理 UNIQUE 冲突。

### 7. drop_index() — 空 → 文件 + 元数据清理

**框架**：两个 `drop_index` 重载都是空的。

**参考实现**：关闭句柄 → 删除文件 → 从 ihs_ 移除 → 从 tab_meta.indexes 移除 → flush_meta。

**为什么有两步清理**：索引有两部分——磁盘上的 .idx 文件（通过 IX 管理）和内存中的元数据（在 TabMeta::indexes 和 ihs_ 中）。必须两边都清理，否则下次 `open_db` 会尝试打开一个已经不存在的索引文件。

## 元数据结构优化

### TabMeta

| 方面 | 框架 | 参考实现 | 为什么改 |
|------|------|---------|----------|
| `indexes` 类型 | `vector<IndexMeta>` | `unordered_map<string, IndexMeta>` | O(n) → O(1) 索引名查找 |
| `cols_map` | 无 | `unordered_map<string, size_t>` | `get_col` 从 O(n) → O(1) |
| `index_names_map` | 无 | `unordered_map<vector<string>, string>` | 避免重复拼接文件名 |
| `get_index_name` | 无 | 有缓存版本的实现 | 减少字符串拼接次数 |

### IndexMeta

| 方面 | 框架 | 参考实现 | 为什么改 |
|------|------|---------|----------|
| `cols` 类型 | `vector<ColMeta>` | `vector<pair<int, ColMeta>>` | 记录键内偏移量，避免每次重新计算 |
| `cols_map` | 无 | `unordered_map<string, pair<int, ColMeta>>` | 根据字段名快速定位偏移 |

### DbMeta

| 方面 | 框架 | 参考实现 | 为什么改 |
|------|------|---------|----------|
| `tabs_` 类型 | `std::map` | `unordered_map` | 有序查找 O(log n) → 哈希 O(1) |
| `friend class Analyze` | 无 | 有 | Analyze 需要直接读取 tabs_ |

### hash 特化

框架缺少对 `IndexMeta` 和 `vector<string>` 的 hash 特化。使用 `unordered_map` 需要 key 类型支持 hash，这两个特化（`src/system/sm_meta.h:104-130`）是必须的。

## 新增方法

参考实现增加了两个框架中没有的方法：

- **`show_indexs`**：查询表上所有索引。虽然是只读操作，但 SM 作为元数据管理者自然应该提供。
- **`redo_index`**：恢复时重建索引。框架不支持恢复，但参考实现的恢复模块需要这个接口。

上一节：[06-system-interaction.md](./06-system-interaction.md) | 下一节：[08-system-api-reference.md](./08-system-api-reference.md)
