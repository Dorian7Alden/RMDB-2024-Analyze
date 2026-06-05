---
name: commit
description: >
  自动暂存并提交 my-analysis/ 和 .claude/ 的修改。写完文档后 PostToolUse hook
  会自动 git add + commit，也可由用户手动说"提交"激活。需要推送时用户说"推送"。
  强制：正确的提交信息格式、单一原则分组、中文描述。
---

# Git 提交

处理 RMDB 项目的 git 提交流程。只提交 `my-analysis/`、`.claude/`、`CLAUDE.md`、`.gitignore`。

**自动提交**：写完文档后，PostToolUse hook 自动暂存并提交修改。提交信息为基本格式，如需修改可用 `git commit --amend`。

## 工作流

### 第 1 步：暂存修改

```bash
git add my-analysis/ .claude/ CLAUDE.md .gitignore
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
- 按层/模块：`architecture`、`storage`、`record`、`index`、`system`、`executor`
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
- 简述：一行精简，约 50 字符，中文，不加句号
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

- 只提交 `my-analysis/`、`.claude/`、`CLAUDE.md`、`.gitignore`，不提交 `src/` 源代码
- 禁止使用 `--no-verify` 或跳过 hooks
- PostToolUse hook 已处理每次 Write/Edit 后的自动提交+推送
