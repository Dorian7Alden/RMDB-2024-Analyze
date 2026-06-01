---
name: feedback_preserve_pinyin
description: 文档中的生僻字拼音标注是用户有意添加的，维护时不能删除
metadata: 
  node_type: memory
  type: feedback
  originSessionId: f52e54f2-7cdf-4f77-b201-c61463aef25b
---

文档中出现的生僻字拼音标注（如"闩 shuān 锁"）是用户手动添加的，用于帮助读者识别不常见的汉字。维护文档时不能将其删除或视为格式错误。

**Why:** 用户此前已手动为生僻字添加了拼音，被我当成格式错误删掉了。这些标注是有意为之，不是无意义的乱码。

**How to apply:**
- 维护文档时，遇到括号内的拼音标注不要删除
- 如果拼音缺少声调或格式不规范，可以补充修正，但不能删除
- 不确定是否为有意添加时，先问用户
