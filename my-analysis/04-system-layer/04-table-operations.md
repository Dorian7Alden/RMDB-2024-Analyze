# 04. 表操作

表是数据库中存储数据的基本单元。SM 层负责管理表的**定义**（元数据），而实际数据存储则委托给 RM 层。

## 操作总览

| 操作 | 方法 | 作用 |
|------|------|------|
| 建表 | `create_table` | 构建字段元数据 + 委托 RM 创建 .db 文件 |
| 删表 | `drop_table` | 级联删除所有索引 + 委托 RM 删除 .db 文件 |
| 显示表列表 | `show_tables` | 遍历 `db_.tabs_` 输出所有表名 |
| 显示表结构 | `desc_table` | 列出表的字段名、类型 |
| 显示索引列表 | `show_indexs` | 列出表上所有索引及其字段 |

## create_table：建表

`src/system/sm_manager.cpp:248-286`

建表是 SM 层最核心的操作之一，它把用户定义的字段列表转换为可用的数据库表。

```cpp
void SmManager::create_table(const std::string& tab_name,
                             const std::vector<ColDef>& col_defs,
                             Context* context) {
  if (db_.is_table(tab_name)) {
    throw TableExistsError(tab_name);
  }
  // 1. 构建字段元数据，计算偏移量
  int curr_offset = 0;
  TabMeta tab;
  tab.name = tab_name;
  for (auto& col_def : col_defs) {
    ColMeta col = {.tab_name = tab_name,
                   .name = col_def.name,
                   .type = col_def.type,
                   .len = col_def.len,
                   .offset = curr_offset};
    curr_offset += col_def.len;
    tab.cols.emplace_back(col);
  }
  // 2. 构建 cols_map 加速查找
  for (size_t i = 0; i < tab.cols.size(); ++i) {
    tab.cols_map.emplace(tab.cols[i].name, i);
  }
  // 3. 创建记录文件
  int record_size = curr_offset;
  rm_manager_->create_file(tab_name, record_size);
  db_.tabs_[tab_name] = std::move(tab);
  // 4. 打开记录文件，缓存句柄
  fhs_.emplace(tab_name, rm_manager_->open_file(tab_name));
  // 5. 持久化元数据
  flush_meta();
}
```

### 第一阶段：构建字段元数据

**输入**：`ColDef` 数组（来自 SQL 解析）。`ColDef` 只含 name、type、len。

**输出**：`ColMeta` 数组，多了 `offset` 和 `tab_name` 字段。

```cpp
// src/system/sm_manager.h:21-25
struct ColDef {
  std::string name;  // Column name
  ColType type;      // Type of column
  int len;           // Length of column
};
```

`ColDef` 是用户输入（来自 SQL），`ColMeta` 是系统内部表示。两者的区别在于 `ColMeta` 多了 `offset`（记录内偏移量）和 `tab_name`（所属表名）。

**偏移量计算示例**：

`CREATE TABLE student (id INT, name STRING(32), age INT)`：

```
遍历字段  curr_offset  字段    len
────────────────────────────────────
初始       0
id         0 → 4       id      4    (offset=0)
name       4 → 36      name    32   (offset=4)
age        36 → 40     age     4    (offset=36)

record_size = 40 字节
```

**含义**：`record_size` 等于各字段长度的总和。因为是定长记录，不需要填充对齐——每条记录严格按字段长度之和分配空间。

### 第二阶段：构建 cols_map

```cpp
for (size_t i = 0; i < tab.cols.size(); ++i) {
  tab.cols_map.emplace(tab.cols[i].name, i);
}
```

`cols_map` 是一个哈希表，从字段名映射到该字段在 `cols` 数组中的下标。

**为什么需要**：Analyze 阶段解析 SQL 时，对 `SELECT name FROM student` 中的 `name`，需要确认 student 表确实有这个字段。`get_col("name")` 通过 `cols_map` 做 O(1) 查找。

**框架对比**：框架没有这段代码，`get_col` 只能遍历 `cols` 做 O(n) 查找。对于列数多的宽表，每次列引用都做线性扫描会累积可观的性能开销。

### 第三阶段：创建记录文件

```cpp
rm_manager_->create_file(tab_name, record_size);
```

**含义**：委托 `RmManager` 创建名为 `tab_name` 的 `.db` 文件，`record_size` 告诉 RM 每条记录多大。

RM 层会在文件第 0 页写入 `RmFileHdr`（记录 `record_size` 等信息），后续插入的记录就按这个大小来分配空间。

### 第四阶段：缓存句柄

```cpp
fhs_.emplace(tab_name, rm_manager_->open_file(tab_name));
```

刚创建的文件立即可用——`open_file` 返回 `RmFileHandle`，存入 `fhs_`。

之后执行器做 INSERT 时，通过 `sm_manager->fhs_["student"]` 就能拿到句柄。

## drop_table：删表

`src/system/sm_manager.cpp:293-322`

```cpp
void SmManager::drop_table(const std::string& tab_name, Context* context) {
  if (!db_.is_table(tab_name)) {
    throw TableNotFoundError(tab_name);
  }
  auto& tab_meta = db_.get_table(tab_name);

  // 1. 关闭并删除记录文件
  rm_manager_->close_file(fhs_[tab_name].get());
  rm_manager_->destroy_file(tab_name);

  // 2. 关闭并删除所有索引文件
  for (auto& [index_name, index_meta] : tab_meta.indexes) {
    ix_manager_->close_index(ihs_[index_name].get());
    ix_manager_->destroy_index(index_name);
    ihs_.erase(index_name);
  }

  // 3. 清理内存状态
  fhs_.erase(tab_name);
  db_.tabs_.erase(tab_name);

  // 4. 持久化
  flush_meta();
}
```

**流程**：
1. 关闭并删除表的 `.db` 记录文件
2. 遍历表上的所有索引，逐个关闭并删除 `.idx` 文件，同时从 `ihs_` 移除
3. 从 `fhs_` 和 `db_.tabs_` 中移除表
4. 持久化元数据（此时 `db.meta` 已经不含这张表了）

**级联删除**：删表时自动删除表上的所有索引。这是一个常见的数据库行为——表没了，索引也没有存在的意义。

**框架对比**：框架的 `drop_table()` 是空的。参赛者需要实现上述完整的级联清理流程。

## show_tables：显示表列表

`src/system/sm_manager.cpp:170-185`

```cpp
void SmManager::show_tables(Context* context) {
  RecordPrinter printer(1);
  printer.print_separator(context);
  printer.print_record({"Tables"}, context);
  printer.print_separator(context);
  for (auto& entry : db_.tabs_) {
    auto& tab = entry.second;
    printer.print_record({tab.name}, context);
  }
  printer.print_separator(context);
}
```

**含义**：遍历 `db_.tabs_`，用 `RecordPrinter` 格式化输出每张表的名字。

这个操作是纯查询，不修改任何数据，也不需要持锁（框架中已完整实现）。

## desc_table：显示表结构

`src/system/sm_manager.cpp:224-240`

```cpp
void SmManager::desc_table(const std::string& tab_name, Context* context) {
  TabMeta& tab = db_.get_table(tab_name);
  std::vector<std::string> captions = {"Field", "Type", "Index"};
  RecordPrinter printer(captions.size());
  printer.print_separator(context);
  printer.print_record(captions, context);
  printer.print_separator(context);
  for (auto& col : tab.cols) {
    std::vector<std::string> field_info = {col.name, coltype2str(col.type)};
    printer.print_record(field_info, context);
  }
  printer.print_separator(context);
}
```

**含义**：列出字段名和类型，类似 MySQL 的 `DESC table_name`。

输出示例（student 表）：

```
| Field | Type   |
| id    | INT    |
| name  | STRING |
| age   | INT    |
```

## show_indexs：显示索引列表

`src/system/sm_manager.cpp:192-217`

**含义**：遍历表上的所有索引，输出索引包含的字段列表。

**注意**：框架中没有 `show_indexs` 方法，这是参考实现额外添加的。

## 串联回顾

表操作的核心流程：

```
CREATE TABLE student (id INT, name STRING(32), age INT)
  │
  ├─→ 1. 构建 ColMeta[3]，计算 offset (0, 4, 36)
  ├─→ 2. 构建 cols_map {"id":0, "name":1, "age":2}
  ├─→ 3. rm_manager_->create_file("student", 40)
  ├─→ 4. fhs_["student"] = rm_manager_->open_file("student")
  └─→ 5. flush_meta()  写入 db.meta

DROP TABLE student
  │
  ├─→ 1. 关闭并删除 student.db
  ├─→ 2. 遍历 indexes: 关闭并删除每个 .idx
  ├─→ 3. 清理 fhs_ / ihs_ / db_.tabs_
  └─→ 4. flush_meta()  更新 db.meta
```

SM 层做"决策"（字段名、类型、长度、偏移量），RM 层做"执行"（创建文件、管理页面）。两者分工明确——SM 管元数据，RM 管数据。

上一节：[03-database-operations.md](./03-database-operations.md) | 下一节：[05-index-operations.md](./05-index-operations.md)
