---
name: feedback-git-commits
description: 每次修改文档后自动 git commit，遵循新格式规范，单一原则
metadata:
  type: feedback
---

每次修改完文档后自动 git commit。提交信息格式：

```
action(keyword): short description

- detail-1
- detail-2
- ...
- detail-n
```

- **action**：`doc`（文档）、`feat`（新功能）、`fix`（修复）、`ref`（重构）、`file`（文件操作）等
- **keyword**：概括修改内容的关键词，如 `rename`（重命名）、`move`（移动）、`add`（新增）、`delete`（删除），或层级名如 `record`、`storage`、`executor`
- **short description**：一行精简描述，因为 `git log --oneline` 默认只显示第一行
- **details**：每条一个具体的操作说明，不能大段描述，可以是补充或备注
- 不同 action 的内容必须分开提交，不能一次提交所有修改
- 只提交 `my-analysis/` 和项目配置文件（如 CLAUDE.md、.claude/），不提交源代码修改

**Why:** 用户需要清晰的修改历史，一行摘要快速浏览，details 提供具体变更。

**How to apply:** 每次完成独立修改后立即提交，按 action 类型分类，不同 action 分开提交。
