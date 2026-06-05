# 09. 系统管理总结

系统管理层共 9 个小节，覆盖了元数据管理和 DDL 执行的完整链路：

```
DbMeta → TabMeta → ColMeta / IndexMeta
(元数据层次)     (表定义)     (字段 / 索引定义)

SmManager
(DDL 执行 + 句柄注册)
  ├── 数据库生命周期: create→open→use→close→drop
  ├── 表操作: create_table / drop_table
  ├── 索引操作: create_index / drop_index / redo_index
  └── 查询: show_tables / desc_table / show_indexs
```

## 各模块框架状态与学习要点

| 模块 | 框架状态 | 核心学习点 |
|------|---------|-----------|
| 元数据结构 | 功能简化 | O(1) hash 查找优化、序列化对称性 |
| 数据库生命周期 | 部分完成 | open_db / close_db 完整流程 |
| flush_meta | 有 bug | `std::ios::trunc` 防止文件膨胀 |
| create_table | 不完整 | 偏移量计算 + cols_map 构建 |
| drop_table | **需实现** | 级联删除索引 |
| create_index | **需实现** | 全表扫描 + memcpy 拼键 + B+ 树插入 |
| drop_index | **需实现** | 索引文件和元数据双清 |
| redo_index | **需新增** | 恢复时索引重建 |

## 核心设计思路

1. **元数据层次**：DbMeta → TabMeta → ColMeta / IndexMeta 四层嵌套，通过 `operator<<`/`>>` 序列化到文本文件 `db.meta`
2. **数据库 = 目录**：建库就是建目录，删库就是删目录，简洁直接
3. **表 = 文件**：每张表对应一个 `.db` 文件，每个索引对应一个 `.idx` 文件
4. **SM 是协调者**：SM 自己不操作 Page，所有实际 I/O 都委托给 RM 和 IX 管理器
5. **句柄注册表**：`fhs_` 和 `ihs_` 缓存所有打开的文件句柄，执行器直接使用
6. **create_index 是最复杂的操作**：跨越全表扫描、键构建、B+ 树插入三个领域

## 与 RM 和 IX 的关系

SM 层是 RM 和 IX 的"上层建筑"：

```
SM（决策层）   → 管理元数据，决定"有什么"
RM（执行层）   → 管理记录，执行"存数据"
IX（执行层）   → 管理索引，执行"查数据"

SM 调用 RM 创建/删除表文件
SM 调用 IX 创建/删除索引文件
执行器通过 SM 获取 RM/IX 句柄 → 直接调用 RM/IX 方法
```

## 与前三章的联系

| 章 | 关系 |
|-----|------|
| 存储层 | SM 通过 `disk_manager_` 创建日志文件，通过 `buffer_pool_manager_` 获取缓冲池 |
| 记录层 | SM 通过 `rm_manager_` 创建/打开/关闭表文件，通过 `RmScan` 全表扫描 |
| 索引层 | SM 通过 `ix_manager_` 创建/打开/关闭索引文件，通过 `IxIndexHandle::insert_entry` 构建索引 |

SM 是最接近用户的存储相关模块——它直接处理 `CREATE TABLE`、`CREATE INDEX` 这类 SQL 命令。

上一节：[08-system-api-reference.md](./08-system-api-reference.md)

下一章：第 5 章：查询处理（待编写）
