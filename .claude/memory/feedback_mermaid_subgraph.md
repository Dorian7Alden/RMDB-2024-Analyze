---
name: mermaid-subgraph-for-hierarchy
description: 表示对象从属关系结构时用 mermaid 子图嵌套呈现
metadata: 
  node_type: memory
  type: feedback
  originSessionId: f52e54f2-7cdf-4f77-b201-c61463aef25b
---

当需要表示对象之间的从属关系、包含关系、层级结构时，用 Mermaid 的 subgraph 子图嵌套来直观呈现。

**Why:** 子图嵌套天然表达"谁包含谁"，比文字列表和表格更直观。

**How to apply:**
- 类 A 包含类 B 的实例 → subgraph A 里面放 subgraph B
- 数组/集合的每个元素是某类型 → subgraph 表示数组，内部子节点表示元素
- 多层嵌套结构 → 层层 subgraph 嵌套
- 子图可以配合 classDef 用颜色区分层级
