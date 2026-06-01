---
name: mermaid-sticky-note-annotations
description: 用特定内容块加箭头指向元素来充当便利贴备注说明
metadata: 
  node_type: memory
  type: feedback
  originSessionId: f52e54f2-7cdf-4f77-b201-c61463aef25b
---

画 Mermaid 图时可以用一个特定样式的内容块，加箭头指向目标元素，充当"便利贴"式的备注说明。

**Why:** 有些补充说明不适合放在节点内部或边标签上，用独立备注块指向目标元素更灵活，像便利贴贴在旁边。

**How to apply:**
- 创建一个特殊样式的节点（如虚线边框、特殊颜色）作为备注块
- 用虚线箭头 `.->` 从备注块指向被说明的目标元素
- 备注块文字可以较长，不受目标节点文字长度限制
- 适合补充"为什么"、"注意"、"前提条件"等说明性文字
