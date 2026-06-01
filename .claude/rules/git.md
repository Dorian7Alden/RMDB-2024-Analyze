# Git 提交规则

## 提交时机

- 每次修改完文档后立即自动 git commit
- 每隔约 10 轮对话，主动检查是否有未提交的修改
- 每次 commit 后立即 `git push`

## 提交信息格式

```
action(keyword): short description

- detail-1
- detail-2
- ...
```

### action 操作类型

| action | 含义 | 适用场景 |
|--------|------|----------|
| `doc` | 文档 | 新增文档、补充内容、调整讲解顺序、修正错误、加图表 |
| `fix` | 修复 | 修正事实性错误、代码引用路径错误、图表数据错误 |
| `ref` | 重构 | 重命名文件、调整目录结构、段落/章节重组、统一格式 |
| `file` | 文件操作 | 移动文件、删除文件、复制文件等纯文件层面变更 |
| `rule` | 规则/配置 | 更新 CLAUDE.md、memory 记忆文件、.claude/ 配置 |

### keyword 关键词

**按层级/模块**：`architecture`、`storage`、`record`、`record-page`、`record-bitmap`、`record-scan`、`record-crud`、`record-free-list`、`index`、`system`、`executor`

**按操作类型**：`rename`、`move`、`add`、`delete`

### short description

- 一行精简，约 50~72 字符，`git log --oneline` 只显示第一行
- 说清楚"做了什么"，中文描述，不加英文
- 不需要句号结尾

### details

- 每条以 `- ` 开头
- 每条一个具体操作，不能大段描述
- 只有一条变更且 short description 已说清楚时，可省略

## 单一原则

**一次提交只做一件事**：
- 不同 action → 必须分开提交
- 不同模块（keyword 不同）→ 建议分开提交
- 逻辑不相关（如"修正 typo"和"新增章节"）→ 必须分开提交

## 提交范围

只提交：`my-analysis/`、`.claude/`、`CLAUDE.md`、`.gitignore`。不提交 `src/` 源代码。

## 来源

合并自 memory 目录下 2 个 git 相关规则文件。提交前读取本文件。
