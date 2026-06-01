---
name: verify-understanding-before-adjusting
description: 用户要求调整内容时先判断理解是否正确，防止盲从错误理解
metadata: 
  node_type: memory
  type: feedback
  originSessionId: f52e54f2-7cdf-4f77-b201-c61463aef25b
---

当用户要求调整内容（如图表、文字等）时，应先判断用户的理解是否正确，再决定是否执行调整。

**Why:** 用户可能基于错误理解提出调整要求。如果不管对错直接执行，会把文档改错，后续还得回滚。这次 LRU 哈希表画成数组就是一个教训。

**How to apply:**
- 用户提出调整 → 先想一下这个调整背后的理解对不对
- 如果用户理解有误 → 先解释清楚正确的理解，确认后再决定怎么改
- 如果用户理解正确 → 正常执行调整
- 核心：保证文档正确性，不盲从指令
