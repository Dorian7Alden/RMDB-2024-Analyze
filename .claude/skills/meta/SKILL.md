---
name: meta
description: >
  Update skills and rules based on user feedback. Use this skill when the user
  gives behavioral feedback — "remember this", "from now on", "always do X",
  "never do Y", "change the rule", "adjust the workflow", "I don't like how you
  did Z", "next time do W". This skill distills the feedback into an actionable
  principle and surgically updates the relevant skill file.
---

# Meta: Skill and Rule Adjustment

This skill handles user feedback about how the model should behave. Instead of
passively recording rules, it actively updates the relevant skill's workflow
steps.

## Workflow

### Step 1: Detect the Feedback Signal

The user is giving behavioral feedback when they say things like:

- "from now on, ..." / "以后..."
- "remember this" / "记住"
- "always do X" / "每次都要..."
- "never do Y" / "不要..."
- "change the rule about Z" / "改一下...的规则"
- "adjust how you..." / "调整..."
- "I don't like how you did Z" / "我不喜欢你..."
- "next time, do W instead" / "下次..."

Any correction or preference about teaching, writing, or workflow behavior is
a trigger.

### Step 2: Identify the Target

Which skill does this feedback affect?

| Feedback about... | Target skill |
|-------------------|-------------|
| Teaching methodology, explanation order, framework comparison, connecting layers | `teach` |
| Document format, writing style, diagrams, navigation, self-review | `write` |
| Git commits, commit messages, staging, pushing | `commit` |
| How rules/skills are managed, this meta process itself | `meta` |

Which specific step in the workflow should change? Read the target skill
file to find the right step.

### Step 3: Distill the Principle

Do NOT transcribe the user's exact words verbatim. Instead:

- Extract the **actionable core**: what should the model DO differently?
- Generalize beyond the specific example: does this apply to ALL teaching,
  or just to a specific scenario?
- If the new instruction contradicts an existing step: **REPLACE** the old one.
  Don't accumulate conflicting rules.
- If it's an addition: add a sub-bullet to the relevant step, or add a new step
  only if it's a distinct phase of the workflow.
- Keep skills concise: prefer refining existing steps over adding new ones.

### Step 4: Update the Skill File

- Read the target skill file
- Make the surgical edit to the specific step
- If the user's instruction was imprecise, refine it while preserving their intent
- Keep the skill self-contained — don't add cross-references to other skills

### Step 5: Confirm

Tell the user exactly what changed:

```
I've updated the [skill name] skill.
Step [N] ([step name]): [what changed].
Before: [old behavior]
After: [new behavior]
```

If you refined the user's original wording, explain how and why.

## Principles

- **Surgical edits**: Change only what needs changing. Don't rewrite the entire skill.
- **Replace, don't accumulate**: If a new rule contradicts an old one, the new one wins.
- **Actionable over abstract**: "Show code before explaining" is better than "源码优先".
- **Keep skills lean**: If adding something, consider if something else can be removed.
