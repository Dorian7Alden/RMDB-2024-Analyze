---
name: beginner-agent-for-maintain
description: 维护文档时须启动新人模拟 agent 提问，发现内容可理解性问题
metadata:
  type: feedback
---

维护文档时，在做完格式/交叉引用等机械检查后，必须启动一个 agent（subagent_type=general-purpose）专门模拟新人阅读文档并提问。该 agent 的任务是：逐文件阅读，站在 DBMS 初学者的角度，对任何感到疑惑、突然出现未解释的概念、逻辑跳跃、缺少实例的地方提出具体问题。

**Why:** 维护者（我）对代码和文档已经很熟悉，容易跳过"新手困惑点"。让另一个 agent 以新人视角提问，能身临其境地发现内容可理解性问题。

**How to apply:** 每次维护文档时，在第 2 步（逐文件检查）和第 3 步（修复并提交）之间插入这个 agent 提问环节。将 agent 提出的问题逐个评估，确实会造成新人困惑的立即修复，不需要的简要说明原因。最后汇报时说明 agent 提出了几个问题、修复了几个。

See also: [[maintain-skill]]
