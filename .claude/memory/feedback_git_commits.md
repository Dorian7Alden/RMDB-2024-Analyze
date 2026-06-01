---
name: feedback-git-commits
description: 每次修改文档后自动 git commit，遵循 action(keyword): description 格式，按操作类型单一提交
metadata:
  type: feedback
---

每次修改完文档后自动 git commit。

## 提交信息格式

```
action(keyword): short description

- detail-1
- detail-2
- ...
- detail-n
```

示例：

```
doc(record-page): 补充 RmPageHandle 构造函数详解和 reinterpret_cast 说明

- page 指针指向当前操作的单页，不是所有页的数组
- reinterpret_cast 将原始 char* 内存重新解释为 RmPageHdr 类型指针
- 用具体数值 0x1000 追踪三个指针的偏移计算过程
```

## 各部分说明

### action 操作类型

必选，表示本次修改的**性质分类**。一次提交只包含同一种 action，不同 action 必须分开提交。

| action | 含义 | 适用场景 |
|--------|------|----------|
| `doc` | 文档 | 新增文档、补充内容、调整讲解顺序、修正错误、加图表 |
| `feat` | 新功能 | （当前项目不涉及源码，仅 my-analysis 文档，一般不用） |
| `fix` | 修复 | 修正文档中的事实性错误、代码引用路径错误、图表数据错误 |
| `ref` | 重构 | 重命名文件、调整目录结构、段落/章节重组、统一格式风格 |
| `file` | 文件操作 | 移动文件、删除文件、复制文件等纯文件层面的变更 |
| `rule` | 规则/配置 | 更新 CLAUDE.md、memory 记忆文件、.claude/ 配置 |

### keyword 关键词

必选，用一个词**概括修改覆盖的范围或操作**，让读者不看 details 也能大致知道涉及什么。可以是层级名、模块名、文件名、或操作类型词。

**按层级/模块**（适合内容修改）：
- `architecture` — 整体架构概览
- `storage` — 存储层
- `record` — 记录层
- `index` — 索引层
- `system` — 系统管理层
- `executor` — 执行器

**按子模块**（更精细时使用）：
- `record-page`、`record-bitmap`、`record-scan`、`record-crud`、`record-free-list`

**按操作类型**（适合文件层面的操作）：
- `rename` — 重命名文件或目录
- `move` — 移动文件位置
- `add` — 新增文件
- `delete` — 删除文件

### short description 简短描述

必选，**一行**概括本次修改的核心内容。约束：
- 必须精简，因为 `git log --oneline` 默认只显示第一行（约 50~72 字符为宜）
- 说清楚"做了什么"，不需要说"为什么"（why 放在 details 里）
- 中文描述，不加英文翻译
- 不需要句号结尾

### details 详细说明

可选但推荐，每行一条具体变更。约束：
- 每条以 `- ` 开头（GitHub/IDE 会自动渲染为列表）
- 每条描述一个独立的具体操作，不能是大段描述
- 可以是补充说明或备注
- 如果只有一条变更且 short description 已经说清楚了，可以省略 details

### 结尾签名

固定追加署名行（和项目无关，仅标记 AI 辅助提交）。

## 提交的单一原则

**一次提交只做一件事**。判断标准：
- 如果两个修改的 action 类型不同 → 必须分开提交
- 如果两个修改涉及不同模块（keyword 不同）→ 建议分开提交
- 如果两个修改逻辑上不相关（如"修正 typo"和"新增章节"）→ 必须分开提交

反例：不能把"重命名文件"、"修正公式错误"、"补充新章节"混在一个 commit 里。

正例：
```
commit 1: doc(record-page): 补充数据页内部布局的 bitmap 位槽对应示例
commit 2: fix(record-page): 修正 bitmap slot 映射图中 b7/s0 对应关系
commit 3: file(rename): 重命名 08→09 09→10 顺延编号，插入新 08 实例串讲
```

## 提交范围

只提交以下目录和文件：
- `my-analysis/` 下的所有文档
- `.claude/` 下的配置和记忆文件
- `CLAUDE.md`
- `.gitignore`

不提交源代码（`src/`）的修改。

**Why:** 清晰的提交历史是项目的"操作日志"。action 区分操作性质，keyword 标注影响范围，一行摘要快速浏览，details 提供变更清单。按单一原则分类后，每条 commit 的 diff 都聚焦一个主题，方便 review 和回退。

**How to apply:** 每完成一个或多个**同类型**的独立修改后，按 action 分类分批提交。不同 action 绝对不混在一起。
