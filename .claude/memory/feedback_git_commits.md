---
name: feedback-git-commits
description: 每次修改文档后自动 git commit，遵循单一原则，不同内容分开提交
metadata: 
  node_type: memory
  type: feedback
  originSessionId: f09191ab-bbf7-4940-9fd0-e5fb21be86df
---

每次修改完文档之后，使用 git 记录修改。要求：
- 不同内容分开提交，遵循单一原则（不要一次性提交大量不相关的修改）
- 提交信息用中文，清楚描述改了什么
- 自动提交，方便回退和 diff 查看
- 只提交 `my-analysis/` 和项目配置文件（如 CLAUDE.md），不提交源代码修改

Why: 用户需要清晰的修改历史记录，方便追溯和回退。
How to apply: 每次完成一个独立的文档修改后，立即 `git add` 对应文件并 `git commit`。
