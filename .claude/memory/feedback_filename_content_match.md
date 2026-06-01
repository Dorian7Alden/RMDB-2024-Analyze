---
name: filename-must-match-content
description: 维护文档时要检查文件名与内容是否匹配，不匹配时一并调整
metadata: 
  node_type: memory
  type: feedback
  originSessionId: f52e54f2-7cdf-4f77-b201-c61463aef25b
---

维护讲解文档时要顺带检查文件名与文档实际内容是否匹配。如果文件名暗示的范围与文档实际覆盖范围不一致，应调整文件名。

**Why:** 文件名是读者第一眼看到的东西。如果文件名说"LRU"但内容覆盖了 LRU、Clock、Replacer 接口等多种内容，读者会被误导或错过。

**How to apply:**
- 维护时看一眼文件名 vs 文档标题和内容范围
- 文件名应反映文档真正的主题范围，不是其中一个子话题
- 改名后同步更新所有引用（README、前后篇链接、交叉引用）
