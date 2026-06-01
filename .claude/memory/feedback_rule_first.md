---
name: feedback_rule_first
description: 用户要求调整/记住规则时，先更新规则再改文档
metadata: 
  node_type: memory
  type: feedback
  originSessionId: f52e54f2-7cdf-4f77-b201-c61463aef25b
---

当用户要求调整规则或记住规则时，**先更新规则文件**（CLAUDE.md 或 memory），再修改文档内容。顺序不能反。

**Why:** 规则是后续所有行为的约束。如果先改文档再记规则，中间那一次改动已经是在"无规则约束"下完成的，可能不符合新规则的要求。规则优先保证了每一次改动都受规则约束。

**How to apply:**
- 用户说"记住……"或"调整一下规则……"时，第一步是更新 CLAUDE.md 或 memory
- 第二步才是按新规则去修改文档
