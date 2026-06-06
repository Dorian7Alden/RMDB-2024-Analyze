---
name: commit
description: >
  提交仓库中除源代码外的所有修改。Hook 不直接执行 git——Hook 发信号，
  Claude 调用本 skill 执行提交。需要推送时用户说"推送"。
  强制：正确的提交信息格式、单一原则分组、中文描述。
---

# Git 提交

处理 RMDB 项目的 git 提交流程。

## 提交范围

**黑名单模式**：提交所有修改，只排除以下源代码目录：

- `src/` — 参考实现源码
- `db2026-x/` — 题目框架源码

```bash
git add -A
git reset -- src/ db2026-x/
```

## Hook 原则

**Hook 禁止直接执行 git 命令。** Hook 只负责两件事：

1. **标记/计数** — 维护标记文件或计数器
2. **发信号** — 通过 `additionalContext` 注入上下文，通知 Claude 调用本 skill

Claude 收到信号后，调用本 skill 按规则执行完整提交流程。

**原因**：Hook 中的 git 命令会绕过提交信息规范、单一原则、分组逻辑，产生大量 `doc(architecture): 更新文档` 这类无意义提交信息。

**信号机制**：

| 触发条件 | 信号 | 动作 |
|---------|------|------|
| 一轮对话中有 Write/Edit | Stop hook → additionalContext | 调用 commit skill（仅提交） |
| 每 5 次对话 | SessionStart → additionalContext | 调用 commit skill（提交 + 推送） |

## 工作流

### 第 1 步：暂存修改

```bash
git add -A
git reset -- src/ db2026-x/
```

查看暂存了哪些文件：
```bash
git diff --cached --name-only
```

如果没有任何暂存文件，报告"没有需要提交的修改"并停止。

### 第 2 步：分类

确定每个逻辑变更组的 `action` 和 `keyword`：

**action**（什么类型的变更）：

| action | 何时使用 |
|--------|---------|
| `doc` | 新增文档、补充内容、调整讲解顺序、修正错误、加图表 |
| `fix` | 修正事实性错误、代码引用路径错误、图表数据错误 |
| `ref` | 重命名文件、调整目录结构、段落/章节重组、统一格式 |
| `file` | 移动文件、删除文件、复制文件等纯文件层面操作 |
| `rule` | 更新 skill、CLAUDE.md、memory 文件、.claude/ 配置 |

**keyword**（哪个领域）：
- 按层/模块：`architecture`、`storage`、`record`、`index`、`system`、`transaction`、`executor`
- 按操作类型：`rename`、`move`、`add`、`delete`

### 第 3 步：单一原则提交

**一次提交只做一件事。** 拆分条件：

- 不同 `action` → 必须分开提交
- 不同 `keyword` → 建议分开提交
- 逻辑不相关（比如"修正 typo"和"新增章节"）→ 必须分开提交

提交信息格式：
```
action(keyword): 中文简述（约 50 字符）

- 细节 1
- 细节 2
```

规则：
- 简述：一行精简，约 50 字符，中文，不加句号。**简述必须概括本次修改的具体内容**（如"补充 B+ 树插入与分裂流程"），不能写"更新文档"这类无意义描述
- 细节：每条以 `- ` 开头，每条一个具体操作
- 简述已说清时可省略细节

示例：
```
doc(index): 补充 B+ 树插入与分裂流程

- 新增 04b-btree-insert.md
- 包含查找叶子节点、插入键值对、节点分裂的完整流程
```

用 heredoc 传提交信息：
```bash
git commit -m "$(cat <<'EOF'
action(keyword): 简述

- 细节
EOF
)"
```

### 第 4 步：推送

```bash
git push
```

完成后告诉用户"已提交并推送"。

## 注意事项

- 禁止使用 `--no-verify` 或跳过 hooks
- 排除 `src/` 和 `db2026-x/` 源代码，其余所有修改均在提交范围内
