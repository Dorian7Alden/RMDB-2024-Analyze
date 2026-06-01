---
name: feedback-auto-persist-rules
description: 用户每次给出的调整规则应自动识别并持久化，防止下次遗忘
metadata: 
  node_type: memory
  type: feedback
  originSessionId: f09191ab-bbf7-4940-9fd0-e5fb21be86df
---

用户每次给出的调整要求、反馈意见中，凡是可以复用于后续工作的，都应自动识别为"规则"。

识别标准：
- 用户明确说"规则"、"记住"、"每次都要"、"以后都"等关键词
- 用户纠正了我的做法（"不要"、"禁止"、"应该"）
- 用户给出了可复用的工作模式或偏好

持久化方式：
- 写入项目根目录 `CLAUDE.md`（每次会话自动加载）
- 写入 `~/.claude/projects/.../memory/` 目录（独立记忆文件）
- 更新 `MEMORY.md` 索引入口

Why: 用户不希望重复交代相同的事情，规则应该一次交代、持久生效。
How to apply: 每次收到用户反馈后，主动判断是否需要持久化为规则，不要等用户提醒。
