# API 速查

## Query

**位置**：`src/analyze/analyze.h:23-60`。

**含义**：`Query` 是 Analyze 输出给 Optimizer 的语义化查询对象。

| 字段 | 类型 | 说明 |
|------|------|------|
| `parse` | `shared_ptr<ast::TreeNode>` | 原始 AST 根节点 |
| `conds` | `vector<Condition>` | WHERE 条件 |
| `group_bys` | `vector<TabCol>` | GROUP BY 列 |
| `havings` | `vector<Condition>` | HAVING 条件 |
| `sort_bys` | `TabCol` | ORDER BY 列 |
| `cols` | `vector<TabCol>` | SELECT 投影列 |
| `agg_types` | `vector<AggType>` | 每个投影列的聚合类型 |
| `alias` | `vector<string>` | 每个投影列的别名 |
| `tables` | `vector<string>` | FROM 中的表名 |
| `set_clauses` | `vector<SetClause>` | UPDATE 的 SET 子句 |
| `values` | `vector<Value>` | INSERT 的值列表 |
| `asc` | `bool` | ORDER BY 或 MIN、MAX 优化中的扫描方向 |
| `limit` | `int` | LIMIT 值，`-1` 表示无限制 |

## Analyze

**位置**：`src/analyze/analyze.h:62-94`。

**含义**：`Analyze` 是语义分析器，把 Parser 生成的 AST 转换为 Query。

| 方法 | 作用 |
|------|------|
| `do_analyze(root)` | 入口方法，按 AST 语句类型分发处理 |
| `check_column(tables, target)` | 验证列存在并补全表名 |
| `get_clause(sv_conds, conds)` | AST 条件转换为 `Condition` |
| `get_having_clause(sv_conds, conds)` | HAVING 条件转换为 `Condition` |
| `check_clause(conds, tables)` | 检查条件中的列存在性和类型兼容性 |
| `convert_sv_value(sv_val)` | AST 字面量转换为 `Value` |
| `convert_sv_comp_op(op)` | AST 比较运算符转换为 `CompOp` |

## PlanTag

**位置**：`src/optimizer/plan.h:23-50`。

**含义**：`PlanTag` 标识计划节点或命令类型。

| 分类 | PlanTag |
|------|---------|
| 工具命令 | `T_Help`、`T_ShowTable`、`T_ShowIndex`、`T_DescTable`、`T_SetKnob` |
| DDL | `T_CreateTable`、`T_DropTable`、`T_CreateIndex`、`T_DropIndex` |
| DML | `T_Insert`、`T_Update`、`T_Delete`、`T_select` |
| 事务 | `T_Transaction_begin`、`T_Transaction_commit`、`T_Transaction_abort`、`T_Transaction_rollback` |
| 执行计划 | `T_SeqScan`、`T_IndexScan`、`T_NestLoop`、`T_SortMerge`、`T_Sort`、`T_Projection`、`T_Aggregate` |
| 其他 | `T_CreateStaticCheckpoint`、`T_Load` |

## Plan 节点

**含义**：Plan 节点是 Optimizer 输出的执行计划，每个节点描述一个操作，但不直接执行。

| 节点 | 位置 | 核心字段 | 作用 |
|------|------|----------|------|
| `ScanPlan` | `src/optimizer/plan.h:61-86` | `tab_name_`、`conds_`、`index_col_names_` | 表扫描计划 |
| `JoinPlan` | `src/optimizer/plan.h:88-109` | `left_`、`right_`、`conds_` | 两个子计划连接 |
| `ProjectionPlan` | `src/optimizer/plan.h:111-129` | `subplan_`、`sel_cols_`、`alias_`、`limit_` | 投影和 LIMIT |
| `SortPlan` | `src/optimizer/plan.h:131-146` | `subplan_`、`sel_col_`、`is_desc_` | 排序 |
| `AggregatePlan` | `src/optimizer/plan.h:148-169` | `sel_cols_`、`agg_types_`、`group_bys_`、`havings_` | 聚合和分组 |
| `DMLPlan` | `src/optimizer/plan.h:186-207` | `subplan_`、`tab_name_`、`values_`、`set_clauses_` | DML 语句包装 |
| `DDLPlan` | `src/optimizer/plan.h:209-225` | `tab_name_`、`tab_col_names_`、`cols_` | DDL 语句包装 |
| `OtherPlan` | `src/optimizer/plan.h:227-238` | `tab_name_` | 工具命令包装 |
| `SetKnobPlan` | `src/optimizer/plan.h:240-251` | `set_knob_type_`、`bool_value_` | 运行时开关 |

## Optimizer

**位置**：`src/optimizer/optimizer.h:24-72`。

**含义**：`Optimizer` 是计划生成的外层入口。

| 方法 | 作用 |
|------|------|
| `plan_query(query, context)` | 简单命令直接生成 Plan，复杂语句交给 Planner |

## Planner

**位置**：`src/optimizer/planner.h:33-64`。

**含义**：`Planner` 是生成 SELECT、INSERT、DELETE、UPDATE 和 DDL 计划的核心类。

| 方法 | 作用 |
|------|------|
| `do_planner(query, context)` | Planner 总入口 |
| `logical_optimization(query, context)` | 逻辑优化，如常量传播 |
| `physical_optimization(query, context)` | 物理优化，如扫描方式和排序方式选择 |
| `make_one_rel(query, context)` | 构建单表扫描或多表连接计划 |
| `generate_sort_plan(query, plan)` | 根据 ORDER BY 包装 SortPlan |
| `get_index_cols(tab_name, curr_conds, index_col_names)` | 判断条件是否可使用索引 |
| `pop_conds(conds, tab_name, solvable_conds, context)` | 从条件列表中取出属于某张表的条件 |

## AbstractExecutor

**位置**：`src/execution/executor_abstract.h:18-59`。

**含义**：`AbstractExecutor` 是所有执行算子的统一接口。

| 方法 | 作用 |
|------|------|
| `beginTuple()` | 初始化算子并定位到第一条结果 |
| `nextTuple()` | 前进到下一条结果 |
| `is_end()` | 判断是否没有更多结果 |
| `Next()` | 返回当前结果记录 |
| `rid()` | 返回当前记录的 RID |
| `cols()` | 返回输出记录的列元数据 |
| `tupleLen()` | 返回输出记录长度 |
| `getType()` | 返回算子类型名 |
| `get_col_offset(target)` | 查找目标列偏移信息 |

## Executor 算子

**含义**：Executor 是真正执行 Plan 的对象，所有算子都继承自 `AbstractExecutor`。

| 算子 | 位置 | 作用 |
|------|------|------|
| `SeqScanExecutor` | `src/execution/executor_seq_scan.h` | 全表扫描并过滤条件 |
| `IndexScanExecutor` | `src/execution/executor_index_scan.h` | 使用 B+ 树索引范围扫描 |
| `NestedLoopJoinExecutor` | `src/execution/executor_nestedloop_join.h` | 嵌套循环连接 |
| `SortMergeJoinExecutor` | `src/execution/executor_sortmerge_join.h` | 排序归并连接 |
| `ProjectionExecutor` | `src/execution/executor_projection.h` | 投影列和 LIMIT |
| `SortExecutor` | `src/execution/executor_sort.h` | 排序 |
| `AggregateExecutor` | `src/execution/executor_aggregate.h` | 聚合和 GROUP BY |
| `InsertExecutor` | `src/execution/executor_insert.h` | 插入记录并维护索引 |
| `DeleteExecutor` | `src/execution/executor_delete.h` | 删除记录并清理索引 |
| `UpdateExecutor` | `src/execution/executor_update.h` | 更新记录并维护索引 |

## execution_defs

**位置**：`src/execution/execution_defs.h:18-111`。

**含义**：`execution_defs.h` 存放执行层共享的小型工具结构和函数。

| 名称 | 位置 | 作用 |
|------|------|------|
| `CondOp` | `src/execution/execution_defs.h:18-60` | 带记录偏移量的条件结构 |
| `compare()` | `src/execution/execution_defs.h:62-80` | 类型感知比较 |
| `ix_compare()` | `src/execution/execution_defs.h:82-89` | 多列索引键比较 |
| `add()` | `src/execution/execution_defs.h:91-111` | UPDATE 增量更新 |

## Portal

**位置**：`src/portal.h:54-274`。

**含义**：Portal 是 Plan 树和 Executor 树之间的转换器。

| 方法 | 位置 | 作用 |
|------|------|------|
| `start(plan, context)` | `src/portal.h:64-165` | 根据 Plan 类型构造 PortalStmt |
| `run(portal, ql, txn_id, context)` | `src/portal.h:168-190` | 根据 PortalStmt 类型调用 QlManager |
| `drop()` | `src/portal.h:193-194` | 清理资源的占位接口 |
| `convert_plan_executor(plan, context, gap_mode)` | `src/portal.h:196-273` | 递归转换 Plan 节点为 Executor 节点 |

## QlManager

**位置**：`src/execution/execution_manager.h:31-54`。

**含义**：`QlManager` 是执行管理器，负责真正驱动执行过程。

| 方法 | 位置 | 作用 |
|------|------|------|
| `run_mutli_query(plan, context)` | `src/execution/execution_manager.cpp:91-115` | 执行 DDL |
| `run_cmd_utility(plan, txn_id, context)` | `src/execution/execution_manager.cpp:117-227` | 执行工具命令和事务命令 |
| `select_from(executorTreeRoot, sel_cols, context)` | `src/execution/execution_manager.cpp:229-295` | 驱动 SELECT 并输出结果 |
| `select_fast_count_star(count, sel_col, context)` | `src/execution/execution_manager.cpp:297-340` | 输出 COUNT 快路径结果 |
| `run_dml(exec)` | `src/execution/execution_manager.cpp:342-344` | 执行 INSERT、DELETE、UPDATE |

## 常见调用入口

**含义**：下面这些入口是阅读查询处理源码时最常跳转的位置。

| 想看什么 | 从哪里开始 |
|----------|------------|
| SQL 主流程 | `src/rmdb.cpp:235-260` |
| 语义分析 | `src/analyze/analyze.cpp:23` |
| 计划生成 | `src/optimizer/optimizer.h:33` |
| SELECT 计划 | `src/optimizer/planner.cpp:565` |
| Plan 到 Executor | `src/portal.h:196` |
| SELECT 输出 | `src/execution/execution_manager.cpp:229` |
| DML 执行 | `src/execution/execution_manager.cpp:342` |

上一节：[07-query-processing-frame-vs-reference.md](./07-query-processing-frame-vs-reference.md) | 下一节：[09-query-processing-summary.md](./09-query-processing-summary.md)
