---
name: git-auto-commit-periodic
description: 每 10 轮左右主动检查并提交用户/linter 修改的文档
metadata: 
  node_type: memory
  type: feedback
  originSessionId: c0551c06-6203-4b7b-9e4e-3d0238e049cb
---

每隔约 10 轮对话，主动执行 `git status --short` 检查是否有未提交的修改。如有，分析 diff 后自动 commit。

**Why:** 用户和 linter 会修改 my-analysis/ 下的文档，这些修改应该被及时提交，方便回退和 diff 查看。

**How to apply:**
- 只提交 `my-analysis/` 下的文档修改
- 不同内容分开提交，遵循单一原则
- 提交信息用中文，描述简洁
- 如果无法确定修改意图（如看起来是误操作），先询问用户
- 其他目录的修改先告知用户再决定是否提交
