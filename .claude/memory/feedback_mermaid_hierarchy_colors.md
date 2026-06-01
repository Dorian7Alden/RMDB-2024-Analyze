---
name: feedback_mermaid_hierarchy_colors
description: mermaid 层级子图用不同颜色区分，层次越深颜色越突出
metadata: 
  node_type: memory
  type: feedback
  originSessionId: f52e54f2-7cdf-4f77-b201-c61463aef25b
---

用 mermaid subgraph 嵌套表示层级从属关系时，每一层使用不同的颜色区分，让层次结构一目了然。配色原则：
- 最外层：淡色（如浅灰、浅蓝）
- 中间层：逐渐加深或变化色调
- 最内层：最突出的颜色，表示核心关注对象

**Why:** 纯黑白或同色嵌套 subgraph 只靠缩进区分层级，视觉上不明显。用不同颜色后，读者一眼就能看出"谁包含谁"，层次关系不再需要靠脑补。

**How to apply:**
- 每个 subgraph 加 `style` 指令，逐层换色
- 不要所有 subgraph 用同一个颜色
- 颜色选择有逻辑：外层保护范围大用冷色调，内层精确到单页用暖色调，等等
