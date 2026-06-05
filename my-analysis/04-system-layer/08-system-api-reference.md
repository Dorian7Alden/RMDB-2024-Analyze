# 08. API 速查

SM 层核心接口一览，方便快速回顾。

## SmManager

### 数据库生命周期

| 方法 | 说明 |
|------|------|
| `SmManager(dm, bpm, rm, ix)` | 构造函数，注入下四层管理器指针 |
| `bool is_dir(db_name)` | 判断数据库目录是否存在 |
| `void create_db(db_name)` | 创建数据库目录 + db.meta + db.log |
| `void drop_db(db_name)` | 删除整个数据库目录 |
| `void open_db(db_name)` | 加载元数据 + 打开所有表/索引文件 |
| `void close_db()` | 落盘元数据 + 关闭所有文件 + 返回上级目录 |
| `void flush_meta()` | 将 db_ 序列化写入 db.meta |

### 表操作

| 方法 | 说明 |
|------|------|
| `void create_table(tab_name, col_defs, context)` | 构建字段元数据 + 创建 .db 文件 + 缓存句柄 |
| `void drop_table(tab_name, context)` | 级联删除所有索引 + 删除 .db 文件 |
| `void show_tables(context)` | 输出所有表名 |
| `void desc_table(tab_name, context)` | 输出表的字段名和类型 |
| `void show_indexs(tab_name, context)` | 输出表上所有索引及字段 |

### 索引操作

| 方法 | 说明 |
|------|------|
| `void create_index(tab_name, col_names, context)` | 全表扫描 + 构建 B+ 树 + 持久化元数据 |
| `void drop_index(tab_name, col_names, context)` | 关闭/删除 .idx 文件 + 清理元数据 |
| `void drop_index(tab_name, cols, context)` | 同上，输入为 ColMeta 向量 |
| `void redo_index(tab_name, tab_meta, col_names, ix_name, context)` | 恢复时重建索引 |

### Getters

| 方法 | 说明 |
|------|------|
| `BufferPoolManager* get_bpm()` | 获取缓冲池管理器 |
| `RmManager* get_rm_manager()` | 获取记录管理器 |
| `IxManager* get_ix_manager()` | 获取索引管理器 |
| `DiskManager* get_disk_manager()` | 获取磁盘管理器 |

### 公开数据成员

| 成员 | 类型 | 说明 |
|------|------|------|
| `db_` | `DbMeta` | 当前数据库的完整元数据 |
| `fhs_` | `unordered_map<string, unique_ptr<RmFileHandle>>` | 表名 → 记录文件句柄 |
| `ihs_` | `unordered_map<string, unique_ptr<IxIndexHandle>>` | 索引文件名 → 索引文件句柄 |

## ColDef

`src/system/sm_manager.h:21-25`

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `string` | 字段名 |
| `type` | `ColType` | 字段类型（INT / FLOAT / STRING） |
| `len` | `int` | 字段长度（字节） |

**含义**：`ColDef` 是用户输入——来自 SQL 的 `CREATE TABLE` 语句。和 `ColMeta` 的区别是它没有 `offset` 和 `tab_name`。

## ColMeta

`src/system/sm_meta.h:21-42`

| 字段 | 类型 | 说明 |
|------|------|------|
| `tab_name` | `string` | 所属表名 |
| `name` | `string` | 字段名 |
| `type` | `ColType` | 字段类型 |
| `len` | `int` | 字段长度（字节） |
| `offset` | `int` | 记录内偏移量（字节） |

**含义**：`ColMeta` 是系统内部表示——比 `ColDef` 多了 `tab_name`（所属表）和 `offset`（建表时计算）。

## IndexMeta

`src/system/sm_meta.h:44-101`

| 字段 | 类型 | 说明 |
|------|------|------|
| `tab_name` | `string` | 所属表名 |
| `col_tot_len` | `int` | 索引键总长度（所有字段长度之和） |
| `col_num` | `int` | 索引包含的字段数量 |
| `cols` | `vector<pair<int, ColMeta>>` | 字段列表（偏移量, 字段元数据） |
| `cols_map` | `unordered_map<string, pair<int, ColMeta>>` | 字段名 → (偏移量, 字段元数据) |

**含义**：`cols` 中的 `int` 是字段在**索引键**中的偏移量，不同于 `ColMeta.offset`（字段在记录中的偏移量）。

## TabMeta

`src/system/sm_meta.h:133-229`

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `string` | 表名 |
| `cols` | `vector<ColMeta>` | 字段列表 |
| `indexes` | `unordered_map<string, IndexMeta>` | 索引名 → 索引元数据 |
| `cols_map` | `unordered_map<string, size_t>` | 字段名 → cols 下标 |
| `index_names_map` | `unordered_map<vector<string>, string>` | 字段集合 → 索引文件名 |

| 方法 | 说明 |
|------|------|
| `bool is_col(col_name)` | 检查字段是否存在 |
| `iterator get_col(col_name)` | 获取字段元数据（O(1)） |
| `bool is_index(col_names)` | 检查索引是否存在 |
| `IndexMeta& get_index_meta(col_names)` | 获取索引元数据 |
| `string get_index_name(col_names)` | 获取索引文件名（带缓存） |

## DbMeta

`src/system/sm_meta.h:233-283`

| 方法 | 说明 |
|------|------|
| `bool is_table(tab_name)` | 检查表是否存在 |
| `TabMeta& get_table(tab_name)` | 获取表元数据 |
| `void SetTabMeta(tab_name, meta)` | 设置表元数据 |

**私有成员**：`name_`（数据库名）、`tabs_`（`unordered_map<string, TabMeta>`）。

## 文件名常量

`src/common/config.h:59,64`

| 常量 | 值 | 说明 |
|------|-----|------|
| `DB_META_NAME` | `"db.meta"` | 元数据文件名 |
| `LOG_FILE_NAME` | `"db.log"` | 日志文件名 |

## 命名约定

| 命名模式 | 示例 | 说明 |
|----------|------|------|
| 表记录文件 | `student.db` | 直接用表名 |
| 索引文件 | `student_age.idx` | `表名_字段1_字段2.idx` |
| 数据库目录 | `student_db/` | 数据库名即目录名 |

上一节：[07-system-frame-vs-reference.md](./07-system-frame-vs-reference.md) | 下一节：[09-system-summary.md](./09-system-summary.md)
