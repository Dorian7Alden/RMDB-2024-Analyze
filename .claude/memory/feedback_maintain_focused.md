---
name: maintain-focus-on-recent-docs
description: 维护文档时聚焦用户最近查看的文档及相关文档，不全量扫描
metadata: 
  node_type: memory
  type: feedback
  originSessionId: f52e54f2-7cdf-4f77-b201-c61463aef25b
---

当用户说"维护一下文档"时，应聚焦于用户最近查看/打开的文档及相关文档，而非全量扫描所有讲解文档。

**Why:** 用户说"维护文档"通常是因为正在看某些文档时发现了问题，全量扫描消耗太大且不必要的。

**How to apply:**
- 优先检查当前 IDE 打开的文件（`<system-reminder>` 中会提示）
- 检查与当前文件有引用关系的上下游文档
- 除非用户明确说"维护所有讲解文档"，否则不进行全量扫描
- 维护范围：当前文档 + 直接关联的文档（前一篇、后一篇、被引用的文档）
