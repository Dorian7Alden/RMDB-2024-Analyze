---
name: commit
description: >
  Automatically stage, commit, and push changes to my-analysis/ and .claude/.
  Use this skill after writing or editing tutorial documents, when the user says
  "commit", or when uncommitted changes are detected in scope. Enforces: proper
  commit message format, single-principle grouping, Chinese descriptions, and
  automatic push.
---

# Git Auto-Commit

This skill handles the git workflow for the RMDB project. Only commit files in
`my-analysis/`, `.claude/`, `CLAUDE.md`, and `.gitignore`.

## Workflow

### Step 1: Stage Changes

```bash
git add my-analysis/ .claude/ CLAUDE.md .gitignore
```

Then check what's staged:
```bash
git diff --cached --name-only
```

If nothing is staged, report "没有需要提交的修改" and stop.

### Step 2: Classify Changes

Determine the `action` and `keyword` for each logical change group:

**action** (what kind of change):
| action | When to use |
|--------|-------------|
| `doc` | New doc, content addition, explanation reorder, error correction, diagrams |
| `fix` | Factual error fix, wrong file path, wrong line number, diagram data error |
| `ref` | Rename files, restructure directories, reorganize sections, unify formatting |
| `file` | Move, delete, or copy files (pure file operations) |
| `rule` | Update skills, CLAUDE.md, memory files, .claude/ config |

**keyword** (which area):
- By layer: `architecture`, `storage`, `record`, `index`, `system`, `executor`
- By operation: `rename`, `move`, `add`, `delete`

### Step 3: Create Commits (Single Principle)

**One commit = one logical change.** Split commits when:

- Different `action` types → MUST split
- Different `keyword` modules → SHOULD split
- Logically unrelated (e.g., "fix typo" + "add new chapter") → MUST split

Commit message format:
```
action(keyword): short Chinese description (~50 chars)

- detail-1
- detail-2
```

Rules:
- Short description: one line, ~50 chars, Chinese, no period at end
- Details: each line starts with `- `, one specific action per line
- If the short description already says everything, skip details
- Co-Authored-By line is appended automatically

Example:
```
doc(index): 补充 B+ 树插入与分裂流程

- 新增 04b-btree-insert.md
- 包含查找叶子节点、插入键值对、节点分裂的完整流程
```

Use a heredoc for the commit message:
```bash
git commit -m "$(cat <<'EOF'
action(keyword): description

- detail
EOF
)"
```

### Step 4: Push

```bash
git push
```

Confirm to user: "已提交并推送" with the commit message summary.

## Important

- Only commit `my-analysis/`, `.claude/`, `CLAUDE.md`, `.gitignore`.
- Never commit `src/` source code.
- Never use `--no-verify` or skip hooks.
- After every doc change, commit immediately — don't batch unrelated changes.
