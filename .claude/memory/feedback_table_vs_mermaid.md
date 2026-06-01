---
name: feedback_table_vs_mermaid
description: 对比/矩阵/规则类内容用表格更直观，mermaid 不是唯一选择
metadata: 
  node_type: memory
  type: feedback
  originSessionId: f52e54f2-7cdf-4f77-b201-c61463aef25b
---

不是所有内容都适合用 mermaid 画图。表格在以下场景比 mermaid 更直观：
- 对比矩阵（如兼容性规则、特性对照）
- 条件判断表（如"已持有 X，请求 Y，结果 Z"）
- 多维度并列比较

画图前先判断：这个内容用表格是否更一目了然。不要为了画图而画图。

**Why:** 用户指出兼容性矩阵用 subgraph 分块展示不如表格直观——表格天然适合表达"行列交叉得出结果"的矩阵逻辑。

**How to apply:**
- 涉及到矩阵/对照/条件组合时，优先考虑表格
- mermaid 适合流程、层级、时间线等动态内容
- 两者配合使用，而不是互相替代
