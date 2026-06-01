---
name: feedback-single-responsibility
description: 每个文档单一职责，只包含当前主要内容，不额外补充无关内容
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 692ac3a3-ca19-4d0c-a72c-66d10a5d47cc
---

每个讲解文档应保持单一职责，只包含当前主题的主要内容。与正文无关的额外内容不应塞进同一个文档中，应该单独新建文档来编写。

**Why:** 避免文档膨胀、主题分散，读者一次只需要理解一件事。如果一份文档既讲 A 又顺带讲 B，读者认知负担加重，后续维护和引用也变得困难。

**How to apply:** 写完一个文档后自查——这份文档如果用一个标题概括，是否中间夹杂了偏离主题的段落。如果是，抽出来独立成篇，在相关文档中通过链接引用。

Related: [[feedback_prerequisite_scope]] [[feedback_writing_style]]
