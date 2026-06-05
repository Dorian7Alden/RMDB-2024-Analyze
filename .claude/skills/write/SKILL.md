---
name: write
description: >
  Write or edit tutorial documentation in my-analysis/. Use this skill whenever
  the user asks to write documentation, create a tutorial doc, add a section,
  edit a doc, or when the teach skill hands off. This skill enforces: proper
  document planning, three-layer structure, indicator labels, short paragraphs,
  mermaid diagram rules, source code referencing, navigation links, and
  self-review.
---

# Tutorial Document Writing

This skill encodes all writing, formatting, and diagram rules for `my-analysis/`
tutorial docs. Follow these steps in order when writing or editing documents.

## Workflow

### Step 1: Plan the Document

Before writing a single line, plan:

- **Filename**: Follow the pattern `##letter-topic.md` (e.g., `02a-record-page.md`).
  Same topic shares a number, a/b/c letters subdivide it. Main content first (a, b, c),
  supplements after.
- **Location**: Determine which `my-analysis/` subdirectory it belongs in.
- **Scope**: One document = one topic. If this covers multiple topics, split it.
- **Sequence**: Check existing docs in the directory to find the correct number.
- If editing an existing doc: read it fully first, understand current structure.
- For new layers, follow the established pattern: overview → data structures →
  components → interaction → API reference → summary.

### Step 2: Data Structure Overview Diagram

For the first/second document of each layer (data structures doc), create a
layered overview diagram using mermaid subgraph nesting:

- Show containment from outer to inner (e.g., .db file → data page → page_hdr + bitmap + slots)
- Each nesting level gets a different color: outer=cool tones, inner=warm tones
- Label each level briefly so readers see "what contains what" at a glance
- Reference: `my-analysis/02-record-layer/02-record-data-structures.md` is the model

### Step 3: Write Each Component Section

For every component, class, or function, follow this pattern IN ORDER:

1. **含义**: What is it? One short sentence defining the concept.
2. **作用** / **用途**: Why does it exist? What problem does it solve?
3. **场景**: Where is it called from? Which callers use it in what flow?
4. Source code block with `// src/xxx/xxx.cpp:line` on the first line inside the block.
   Do NOT add a separate `**源码**：` label outside the block — the comment inside is sufficient.
5. **示例**: Concrete example with actual data values.
6. **输入** / **输出**: Parameter and return value meanings (for functions/methods).

Section formatting rules:
- **Short paragraphs**: One sentence per paragraph where possible. Break at every period.
- **Indicator labels**: Every section MUST start with a label (含义, 作用, etc.) so readers know what kind of information they're reading.
- **Abbreviations**: Define on first use: "RM (Record Manager)".
- **English terms**: Only include English originals for source code identifiers, not for ordinary words.
- **Key terminology**: Distinguish 键 (key) vs 键值对 (key-value pair). Never say just "键值" — it's ambiguous.
- **Titles**: Topic only, no dashes, parentheses, or English translations in headings.

### Step 4: Add Diagrams Where Helpful

Choose the right format:

- **Use mermaid** for: flowcharts, sequence diagrams, hierarchical relationships.
- **Use tables** for: comparison matrices, conditional logic, multi-dimensional evaluation.
- **Use text file trees** for: project structure, directory layout.

Mermaid rules:
- No `()` `<>` `*` `/` in node text — use text descriptions instead.
- If a diagram needs a color/symbol legend, put the legend in a separate second diagram.
- Use subgraph nesting for containment relationships.
- Colored subgraphs: outer=cool tones, inner=warm tones, deeper=more prominent.
- Use sticky-note style nodes (dashed border, special color) + dotted arrows (`-.->`) for annotations.

### Step 5: Specify Locks Precisely

Whenever mentioning locks, state ALL four dimensions:

- **Level**: Page latch (RLatch/WLatch), transaction lock, root mutex (root_latch_), etc.
- **Scope**: What resource is locked (entire file, single page, single node).
- **Type**: Read vs write, shared vs exclusive.
- **Lifecycle**: When acquired, when released.

Don't just say "it locks". Be specific about what, how, and for how long.

### Step 6: Navigation and Polish

- End every document with navigation links:
  ```
  上一节：[filename.md](./filename.md) | 下一节：[filename.md](./filename.md)
  ```
- Use filenames as the link text, NOT Chinese titles.
- If the document is the first or last in its sequence, only include the applicable direction.

### Step 7: Self-Review

After writing, verify:

- [ ] All file paths and line numbers are accurate
- [ ] No orphan references to concepts not yet explained
- [ ] No long paragraphs that should be split (one sentence = one paragraph)
- [ ] Every abbreviation defined on first use
- [ ] Every section has an indicator label
- [ ] No "键值" ambiguity — use "键" or "键值对" explicitly
- [ ] No special characters in mermaid diagrams
- [ ] Navigation links present at the end

### Step 8: Signal Commit

After writing is complete and self-reviewed, invoke the `commit` skill to stage,
commit, and push the changes.

## Quality Standards

- **Ready to publish**: Consistent style, consistent formatting, no draft-like quality.
- **Self-contained**: Each doc is complete on its own topic. Cross-references are fine but the reader shouldn't need to read three other docs to understand this one.
- **Zero-base friendly**: Assume the reader is a beginner. Supplement necessary prerequisite concepts inline.
