---
name: teach
description: >
  Teach DBMS concepts with proper methodology. Use this skill whenever the user
  asks to learn, understand, explain, or explore any DBMS concept — "teach me X",
  "how does X work", "what is X", "explain X", "walk me through X". This skill
  enforces: context first, connect to previous layers, source code before
  explanation, three-layer structure (overview→detail→synthesis), framework vs
  reference comparison, and concrete examples.
---

# DBMS Teaching Methodology

This skill encodes the teaching methodology for the RMDB project. Follow these
steps in order when the user asks to learn a concept. Each step builds on the
previous one — don't skip.

## Workflow

### Step 1: Establish Context

Before any explanation, orient the reader:

- Which layer/module does this concept belong to? (storage / record / index / system / transaction / execution)
- What is it? (one sentence)
- Why does it exist? (what problem does it solve?)
- Where does it fit in the architecture? (mention the `src/` directory)

Keep this brief — the goal is a mental map, not a textbook chapter.

### Step 2: Connect to Previous Layers

Make the knowledge coherent by linking to what the user already learned:

- **Similarities**: "Like [previous layer]'s X, this Y also..." (e.g., both `.db` and `.idx` are fixed-length page files, both load by Page, both have file headers and page headers)
- **Differences**: "But unlike [previous layer], here Z is different because..." (e.g., Bitmap vs Keys array, free list vs B+ tree structure)
- Use a comparison table when there are 3+ dimensions to compare

This step prevents isolated knowledge — the user should feel each layer builds on the last.

### Step 3: Source Code First

Show the code BEFORE explaining it. The source is the primary material; your explanation is the supplement.

- Open the header file first — show class/struct declarations
- Give the file path and line range
- Let the reader SEE what they're working with
- Only then explain what the code means

Format:
```
**源码**: `src/xxx/xxx.h:10-50`

[code block]
```

### Step 4: Three-Layer Explanation

This is the core of the teaching method. Never start with a specific method or code snippet — always go top-down.

**Layer 1 — Abstract Overview:**
Give the big picture. What's the whole thing about? Use a mermaid flowchart or a text summary. The reader should understand: input, output, phases, and how phases relate.

**Layer 2 — Block-by-Block Detail:**
Take each phase from the overview and dive deep. For each block, follow the same mini-loop: what this block does → show the code → explain the logic → give an example. One block at a time, in order.

**Layer 3 — Connect Back:**
After all blocks are explained, return to the overview perspective. Show the call chain, data flow direction, dependency hierarchy. The reader should now have a complete mental model.

Never skip Layer 1. The reader must know the whole before understanding the parts.

### Step 5: Framework Contrast

Compare `db2026-x/` (competition framework — what the learner needs to write) vs `src/` (reference implementation — how it's done):

- For each difference, explain **WHY** the reference made that choice, not just WHAT changed
- Use a comparison table: | Aspect | Framework (db2026-x) | Reference (src) | Why Different |
- Framework code that's already complete gets a brief mention only — focus on TODO/empty method bodies
- The goal: the learner understands the design rationale, not just copies code

### Step 6: Concrete Walkthrough

Abstract explanations aren't enough. Walk through a specific example end-to-end:

- Use a concrete table (e.g., "student table with id INT, name STRING, age INT")
- Use specific values (e.g., "insert age=20 into node (18, 22, 25)")
- Trace the data flow step by step: "when X calls Y, here's what happens"
- Show before/after states

### Step 7: Scope Assessment

After the explanation, assess what documentation this topic needs:

- Does this layer have an interaction doc? (each layer must have one — check `my-analysis/0X-.../`)
- Does this layer have an API reference doc? (each layer must have one)
- Does this topic need a data structure overview diagram? (first doc of each layer must have one)
- Are there related concepts the user asked about repeatedly that should become standalone docs?

Tell the user what's available and what might be worth creating.

### Step 8: Handoff

Ask the user: "Should I write this up as a tutorial document?"

- If yes → invoke the `write` skill to create/edit the document
- If no → stop here. The teaching is complete.

## Key Principles

- **What before why before where**: Define the concept → explain the motivation → show where it's used
- **Source code is primary**: The ultimate goal is to read and write source code. Every explanation must trace back to actual code.
- **Concrete over abstract**: Every abstract concept gets a concrete example.
- **Respect the framework**: The learner's task is to fill in `db2026-x/`. Help them understand the design, not just copy `src/`.
