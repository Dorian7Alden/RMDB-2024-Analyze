# 框架与参考实现对比

## 总体概览

**含义**：本节对比 `db2026-x/src/` 框架和 `src/` 参考实现，说明查询处理模块中哪些部分是学习重点。

**作用**：框架已经给出 Parser 和一部分上层骨架，真正需要掌握的是 Analyze、Optimizer、Execution 和 Portal 中的空实现与高级扩展。

| 模块 | 框架状态 | 参考实现 | 学习重点 |
|------|----------|----------|----------|
| Parser | 基本提供 | 功能更完整 | 理解输入输出即可 |
| Analyze | 部分 TODO | 完整语义检查 | 表检查、UPDATE、列检查 |
| Optimizer | 基础计划生成 | 增强索引选择和聚合计划 | 常量传播、最长前缀匹配、聚合包装 |
| Portal | 基础转换 | 支持聚合、排序归并连接、写间隙模式 | Plan 到 Executor 的映射 |
| Execution | 多数算子为空 | 多数算子完整 | 查询处理核心实现 |
| QlManager | 基础输出 | 增加快路径和检查点处理 | 执行分发和结果输出 |

## Analyze

### 表存在性检查

**位置**：`db2026-x/src/analyze/analyze.cpp:25` 标注 `TODO: 检查表是否存在`。

**现状**：框架把 `SelectStmt` 中的表名移动到 `query->tables` 后，没有验证这些表是否真的存在。

```cpp
// db2026-x/src/analyze/analyze.cpp:21-25
if (auto x = std::dynamic_pointer_cast<ast::SelectStmt>(parse))
{
    // 处理表名
    query->tables = std::move(x->tabs);
    /** TODO: 检查表是否存在 */
```

**需要实现**：遍历 `query->tables`，调用 `sm_manager_->db_.is_table(table)` 检查表名，不存在时抛出 `TableNotFoundError`。

**参考位置**：`src/analyze/analyze.cpp:28-34`。

### UPDATE 语义分析

**位置**：`db2026-x/src/analyze/analyze.cpp:50-52` 的 `UpdateStmt` 分支为空。

**现状**：框架没有构造 `set_clauses`，没有检查 SET 子句类型，也没有处理 UPDATE 的 WHERE 条件。

```cpp
// db2026-x/src/analyze/analyze.cpp:50-52
} else if (auto x = std::dynamic_pointer_cast<ast::UpdateStmt>(parse)) {
    /** TODO: */

}
```

**需要实现**：先把 AST 中的 SET 子句转换为 `SetClause`，再根据表元数据检查赋值类型，最后调用 `get_clause()` 和 `check_clause()` 处理 WHERE 条件。

**参考位置**：`src/analyze/analyze.cpp:162-195`。

### 显式表名列检查

**位置**：`db2026-x/src/analyze/analyze.cpp:70-90` 的 `check_column()` 分支中，显式表名路径为空。

**现状**：框架能处理 `SELECT age FROM student` 这种省略表名的列引用，但没有完整处理 `SELECT student.age FROM student` 这种显式表名写法。

```cpp
// db2026-x/src/analyze/analyze.cpp:70-90
TabCol Analyze::check_column(const std::vector<ColMeta> &all_cols, TabCol target) {
    if (target.tab_name.empty()) {
        // Table name not specified, infer table name from column name
        // ...
    } else {
        /** TODO: Make sure target column exists */
        
    }
    return target;
}
```

**需要实现**：当 `target.tab_name` 非空时，检查候选列中是否存在同表同列的 `ColMeta`，找不到就抛出 `ColumnNotFoundError`。

**参考位置**：`src/analyze/analyze.cpp:235-248`。

## Optimizer

### 逻辑优化为空

**位置**：`db2026-x/src/optimizer/planner.cpp:120-125` 的 `logical_optimization()` 只有 TODO 后直接返回。

```cpp
// db2026-x/src/optimizer/planner.cpp:120-125
std::shared_ptr<Query> Planner::logical_optimization(std::shared_ptr<Query> query, Context *context)
{
    
    //TODO 实现逻辑优化规则

    return query;
}
```

**现状**：框架不会改写 WHERE 条件，`a = 1 AND b = a` 仍然保持原样。

**需要实现**：实现常量传播，把 `a = 1 AND b = a` 改写成 `a = 1 AND b = 1`，让 `b = 1` 也有机会匹配索引。

**参考位置**：`src/optimizer/planner.cpp:240-266`。

### 索引匹配能力不足

**位置**：`db2026-x/src/optimizer/planner.cpp` 的 `get_index_cols()`。

**现状**：框架的索引选择偏基础，只适合简单的等值条件匹配。

**参考位置**：`src/optimizer/planner.cpp:21-149`。

```cpp
// src/optimizer/planner.cpp:21-23
bool Planner::get_index_cols(std::string& tab_name,
                             std::vector<Condition>& curr_conds,
                             std::vector<std::string>& index_col_names) {
```

**需要实现**：参考实现支持最长前缀匹配、等号条件数量决胜和条件重排。

**示例**：如果索引是 `(a, b, c)`，查询条件是 `a = 1 AND b > 2`，即使没有覆盖 `c`，也可以利用 `(a, b)` 这个前缀范围。

### 高级计划节点缺失

**位置**：参考实现的 Plan 类型定义在 `src/optimizer/plan.h:23-50`。

**现状**：框架虽然有部分 PlanTag，但缺少参考实现中完整连接到执行层的高级节点能力。

| 能力 | 参考位置 | 作用 |
|------|----------|------|
| `AggregatePlan` | `src/optimizer/plan.h:148-169` | 表示聚合和分组计划 |
| `StaticCheckpointPlan` | `src/optimizer/plan.h:253-259` | 表示静态检查点命令 |
| MIN 和 MAX 优化 | `src/optimizer/planner.cpp:280-329` 附近 | 用索引顺序减少聚合代价 |
| 排序归并连接选择 | `src/optimizer/planner.cpp:437-468` 附近 | 根据运行时开关选择连接算法 |

**需要实现**：先完成基础扫描、连接、排序、投影，再学习聚合计划和高级优化，否则容易在多个未完成点之间迷路。

## Portal

### 聚合算子映射

**位置**：参考实现在 `src/portal.h:236-240` 增加了 `AggregatePlan` 到 `AggregateExecutor` 的转换。

```cpp
// src/portal.h:236-240
if (auto x = std::dynamic_pointer_cast<AggregatePlan>(plan)) {
  return std::make_unique<AggregateExecutor>(
      convert_plan_executor(x->subplan_, context), std::move(x->sel_cols_),
      std::move(x->agg_types_), std::move(x->group_bys_),
      std::move(x->havings_), context);
}
```

**现状**：框架没有 `AggregateExecutor`，所以即使 Optimizer 生成聚合计划，也无法真正执行。

**需要实现**：创建聚合算子后，在 Portal 中补上 Plan 到 Executor 的映射。

### 排序归并连接映射

**位置**：框架 `db2026-x/src/portal.h:168-174` 总是创建 `NestedLoopJoinExecutor`。

**现状**：即使 Planner 生成了 `T_SortMerge`，框架 Portal 仍然不会创建 `SortMergeJoinExecutor`。

**参考位置**：`src/portal.h:260-265`。

```cpp
// src/portal.h:260-265
if (x->tag == T_NestLoop) {
  return std::make_unique<NestedLoopJoinExecutor>(
      std::move(left), std::move(right), std::move(x->conds_));
}
return std::make_unique<SortMergeJoinExecutor>(
    std::move(left), std::move(right), std::move(x->conds_));
```

**需要实现**：先实现 `SortMergeJoinExecutor`，再让 Portal 根据 `JoinPlan::tag` 选择正确算子。

### UPDATE 和 DELETE 的写间隙模式

**位置**：参考实现在 `src/portal.h:104-146` 中为 UPDATE 和 DELETE 的扫描子计划传入 `gap_mode = true`。

**现状**：框架收集 RID 的思路已经有了，但没有参考实现中的写间隙模式和索引键更新优化。

**需要实现**：理解事务层后，再补全扫描阶段的锁语义和 UpdateExecutor 的索引维护优化。

## Execution

### 算子空实现

**含义**：Execution 是框架中待实现最多的部分。

**作用**：这些算子直接决定 SELECT、DELETE、UPDATE 是否真的能跑通。

| 算子 | 框架位置 | 框架状态 | 参考位置 |
|------|----------|----------|----------|
| `SeqScanExecutor` | `db2026-x/src/execution/executor_seq_scan.h:48-57` | 三个核心方法为空 | `src/execution/executor_seq_scan.h:80-111` |
| `IndexScanExecutor` | `db2026-x/src/execution/executor_index_scan.h:67-76` | 三个核心方法为空 | `src/execution/executor_index_scan.h:214-239` |
| `NestedLoopJoinExecutor` | `db2026-x/src/execution/executor_nestedloop_join.h:46-55` | 三个核心方法为空 | `src/execution/executor_nestedloop_join.h:52-119` |
| `ProjectionExecutor` | `db2026-x/src/execution/executor_projection.h:42-47` | 三个核心方法为空 | `src/execution/executor_projection.h:58-87` |
| `SortExecutor` | `db2026-x/src/execution/execution_sort.h:45` | `Next()` 返回 nullptr | `src/execution/executor_sort.h` |
| `DeleteExecutor` | `db2026-x/src/execution/executor_delete.h:40` | `Next()` 返回 nullptr | `src/execution/executor_delete.h:56-91` |
| `UpdateExecutor` | `db2026-x/src/execution/executor_update.h:42` | `Next()` 返回 nullptr | `src/execution/executor_update.h:66-164` |

**学习顺序**：建议按 `SeqScanExecutor`、`ProjectionExecutor`、`NestedLoopJoinExecutor`、`SortExecutor`、`DeleteExecutor`、`UpdateExecutor`、`IndexScanExecutor` 的顺序实现。

**原因**：SeqScan 和 Projection 能先跑通最简单的 SELECT，IndexScan 依赖索引层和谓词边界分析，难度最高。

### InsertExecutor 已实现

**位置**：框架 `db2026-x/src/execution/executor_insert.h` 已经有完整插入逻辑，末尾 `return nullptr` 是 DML 算子的正常返回方式。

**含义**：InsertExecutor 不返回查询结果，它的 `Next()` 负责完成插入副作用。

**注意**：不要把 InsertExecutor 末尾的 `return nullptr` 误判成 TODO。

### 共享工具缺失

**位置**：参考实现的执行层工具在 `src/execution/execution_defs.h:18-111`。

**现状**：框架的 `execution_defs.h` 基本为空，后续算子需要的比较和加法工具不完整。

| 工具 | 参考位置 | 用途 |
|------|----------|------|
| `CondOp` | `src/execution/execution_defs.h:18-60` | 表示带偏移量的条件 |
| `compare()` | `src/execution/execution_defs.h:62-80` | INT、FLOAT、STRING 的类型感知比较 |
| `ix_compare()` | `src/execution/execution_defs.h:82-89` | 多列索引键逐列比较 |
| `add()` | `src/execution/execution_defs.h:91-111` | UPDATE 增量赋值 |

**需要实现**：执行算子中不要直接用 `memcmp` 比所有类型，FLOAT 的字节序和数值大小不等价。

## QlManager

### 输出文件开关

**位置**：参考实现的 `select_from()` 在 `src/execution/execution_manager.cpp:244-252` 和 `src/execution/execution_manager.cpp:281-287` 按 `enable_output_file` 决定是否写 `output.txt`。

**现状**：框架会直接写 `output.txt`，参考实现增加了运行时开关。

**需要实现**：配合 `SET enable_output_file` 使用，避免所有查询都无条件写文件。

### COUNT 快路径

**位置**：参考实现在 `src/rmdb.cpp:246-253` 判断无条件 `COUNT`，并调用 `select_fast_count_star()`。

**现状**：框架没有这条快路径。

**作用**：`COUNT(*)` 不必真的扫出每条记录再计数，可以直接利用表文件头或辅助统计信息返回数量。

### 静态检查点处理

**位置**：参考实现在 `src/execution/execution_manager.cpp:182-226` 处理 `StaticCheckpointPlan`。

**现状**：框架没有完整恢复模块联动时，这部分不是第 5 章优先实现目标。

**学习建议**：等学到第 7 章恢复层时，再回来看这段逻辑。

## 小结

**核心判断**：第 5 章的框架学习重点不是 Parser，而是 Analyze、Optimizer、Portal 和 Execution 的协作。

**最低可运行路径**：表检查、SeqScan、Projection、QlManager 输出跑通后，可以执行最简单的 `SELECT col FROM table WHERE col = value`。

**进阶路径**：再补 IndexScan、Join、Sort、Delete、Update、Aggregate 和 SortMergeJoin，逐步覆盖完整 SQL 能力。

上一节：[06-query-processing-interaction.md](./06-query-processing-interaction.md) | 下一节：[08-query-processing-api-reference.md](./08-query-processing-api-reference.md)
