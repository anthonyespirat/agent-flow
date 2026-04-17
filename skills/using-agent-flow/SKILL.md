---
name: using-agent-flow
description: Use when the user describes a dev task (feature, bugfix, refactor, or anything that will touch code). Establishes the skill-check discipline — before writing code or answering, verify whether writing-plans, executing-plans, or subagent-execution should run first.
---

# Using agent-flow

<SUBAGENT-STOP>
If you were dispatched as a subagent to execute a specific task, skip this skill — you already have your instructions.
</SUBAGENT-STOP>

## The rule

**For any task that will touch code, check the agent-flow skills BEFORE responding or editing.** The default instinct is to read files and start typing. That's the mistake this skill exists to prevent.

If there's even a 1% chance that one of these applies, invoke it:

- **`writing-plans`** — user describes a feature, bugfix, refactor, or multi-step change. Before any code. Triggers on phrases like "add", "build", "fix", "refactor", "implement", a Linear reference (`ENG-123`), or any task that isn't a pure one-liner question.
- **`executing-plans`** — a plan file already exists (`.claude/plan/*.md`) and the user wants it executed in this session.
- **`subagent-execution`** — same as above but the user picked the subagent-driven mode, or wants the main chat kept light.

If none apply, proceed normally.

## Chain

The skills chain themselves — you don't orchestrate:

```
writing-plans ─┬─▶ executing-plans         (user picks [1] in-session)
               └─▶ subagent-execution      (user picks [2] subagent-driven)
```

`writing-plans` ends by asking the user which mode. `executing-plans` and `subagent-execution` end by offering to run the `test-runner` agent. That's the whole flow.

## Red flags — STOP if you think any of these

| Thought | Reality |
|---|---|
| "This is a small change, I'll skip planning" | Small changes become big. Invoke writing-plans — a 3-step plan is cheap and the user can say "skip it, just do X" after seeing it. |
| "Let me read the files first to see what's needed" | That's what codebase-explorer does inside writing-plans. Check the skill first. |
| "I'll just fix the typo / rename / one-liner" | Truly trivial edits (typo, comment fix, single-line rename) can skip. Anything with logic or > 1 file: invoke writing-plans. |
| "The user already has a plan in context, I'll just implement" | Exactly — that's why executing-plans or subagent-execution exists. Pick one. |
| "I can do this faster manually than by dispatching" | Maybe, but you lose the on-disk plan + todo discipline. Use the skill. |

## User instructions override

If the user explicitly says "skip planning, just do X" or "don't use subagents" or similar, obey. Skills are the default; user intent wins.

## Announce before acting

When you invoke one of these skills, say one short line:

> "Using `writing-plans` to draft the plan for this change."

No preamble, no explanation. One line, then run.
