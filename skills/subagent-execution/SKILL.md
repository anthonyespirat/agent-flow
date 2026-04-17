---
name: subagent-execution
description: Use when a plan file at .claude/plan/{slug}.md exists and the user has chosen subagent-driven execution (mode [2] from writing-plans). Reads the plan, builds a todo from its STEPS, then dispatches one fresh general-purpose subagent per STEP with precisely scoped context. The main conversation stays light — you are the controller, not the implementer.
---

# Subagent-driven execution

Execute a plan by dispatching a fresh subagent per STEP. You (the controller) own the todo and the review; the subagents own the edits. This keeps this conversation light and prevents context pollution across steps.

**Announce at start:** "Using `subagent-execution` to implement `<plan path>` — one subagent per step."

## Why subagents

Each subagent gets exactly the context it needs for its step — no more, no less. They don't inherit this conversation's history. You curate their inputs and review their summaries. Your context stays focused on orchestration.

## Input

- Path to the validated plan file (e.g. `.claude/plan/eng-123.md`). If the user didn't give it explicitly, use the most recent file in `.claude/plan/`.

## Process

### Step 1 — Read and critically review the plan

`Read` the plan file. Parse `SKILLS TO APPLY`, `FILES TO CHANGE`, `STEPS`, `RISKS`, `OUT OF SCOPE`.

Review it critically before dispatching anything:

- Ambiguous steps, missing setup, contradictions, references to non-existent symbols
- `QUESTIONS FOR USER` section → plan isn't ready, STOP and report
- Steps that can't be isolated (step 3 can't run without step 2's output in the same subagent context) → flag this; you may need to merge steps when dispatching

If you find concerns, raise them with the user in one message, then wait. Don't silently dispatch.

### Step 2 — Extract all step contexts upfront

For each STEP, write down (keep this in your working memory — do NOT write a new file):

- **Step text** (the `- [ ]` line, imperative)
- **Relevant files** — from `FILES TO CHANGE` that this step touches
- **Applicable guideline skills** — the subset of `SKILLS TO APPLY` that constrains this step
- **Prior step outputs** — if the step depends on work done in an earlier step, note which file(s) were created/modified and by whom

This is the material you'll paste into each dispatch. Extracting upfront means you won't have to re-read the plan per step.

### Step 3 — Build the todo

Call `TodoWrite` once with one task per STEP:

- `content`: the step text
- `activeForm`: present continuous
- `status`: `pending` for all tasks

You (controller) own this todo. You update it as each subagent reports back.

### Step 4 — Dispatch loop

For each step in order:

1. `TodoWrite` → mark the current step `in_progress`.
2. Dispatch a **fresh general-purpose subagent** (`Agent` tool, `subagent_type: general-purpose`) using the template at `./implementer-prompt.md`. Fill in STEP, SCENE, RELEVANT FILES, GUIDELINE SKILLS, and (if applicable) PRIOR-STEP OUTPUTS.
3. Wait for the subagent's report.
4. Handle the status (see Step 5).
5. When the step is accepted: `TodoWrite` → mark `completed`, move to next step.

Never run multiple subagents in parallel for steps that touch overlapping files — conflicts.

### Step 5 — Handle the subagent's status

Every subagent returns one of four statuses. Handle each precisely — never silently ignore a non-DONE status, and never retry the same dispatch unchanged.

**DONE** — step is complete, self-review clean.
1. Verify the report against reality:
   - Did it stay within RELEVANT FILES? If it edited others, is the reason justified in NOTES? If not, re-dispatch with tighter constraints.
   - Did it actually apply the listed GUIDELINE SKILLS? If a skill rule was ignored, re-dispatch with "Must follow <skill> — specifically <rule>".
   - Any forbidden action (commit, push, PR)? → escalate to user immediately.
2. If all checks pass, mark the step `completed` and move on.

**DONE_WITH_CONCERNS** — step is implemented but the subagent flagged doubts. Read the concerns before proceeding.
- If a concern is about **correctness or scope** (e.g. "I'm not sure this handles the null case", "I had to edit a file outside RELEVANT FILES") → address it before moving on: either re-dispatch the same subagent role with targeted feedback, or escalate to the user if the concern points at a plan gap.
- If a concern is an **observation** (e.g. "this file is getting large", "this pattern feels brittle") → note it, don't block on it. Proceed.

**NEEDS_CONTEXT** — the subagent needed information that wasn't in its prompt.
1. Figure out what's missing (the report tells you).
2. Gather it: read an additional file, pull from the plan's APPROACH section, fetch from the explorer context you already have — or ask the user if it's a real gap.
3. Re-dispatch the **same step with the same model**, adding the missing context. Do not change the STEP text; change the surrounding context.

**BLOCKED** — the subagent cannot complete the step. Assess the nature of the blocker:
- **Context gap** (subagent says "I don't have enough info about X") → provide context and re-dispatch.
- **Reasoning gap** (step requires architectural judgment the subagent didn't make) → re-dispatch with a more capable model (if available) or break the step into smaller pieces and dispatch them sequentially.
- **Step too large** (subagent reads file after file without progress) → split the step into 2–3 tighter ones, dispatch each, update your todo.
- **Plan is wrong** (subagent surfaces a contradiction or missing dependency) → STOP. Escalate to the user. The plan needs revision before you can proceed.

**Never** ignore a BLOCKED or NEEDS_CONTEXT and mark the step `completed`. **Never** retry with the same prompt and same model — something has to change (context, model, or step size).

**Escalation budget.** If the same step hits a non-DONE status twice after adjustments, STOP and escalate to the user. Two failed dispatches is a signal that the step framing or the plan itself is the problem, not the subagent.

### Step 6 — Final report

After all steps are `completed`:

```
STATUS: done | blocked | partial

PLAN: <path>

FILES CHANGED (aggregated across subagents):
- path/to/file.ts — <what changed>

STEPS COMPLETED: <N of M>

SUBAGENT DISPATCHES: <count> (<re-dispatches if any>)

NOTES:
- <anything notable from subagent reports>
```

### Step 7 — Offer tests

End with:

> "Implementation done. Run the `test-runner` agent? (scope: backend / frontend / fullstack)"

On failure, dispatch `test-runner`; on its fix-fail loop, send a fresh subagent with the failure report as input. Max 2 fix iterations before escalating.

---

## Prompt template

The dispatch prompt lives in a separate file so it stays curatable:

- `./implementer-prompt.md` — Dispatch implementer subagent

Read it once at the start of Step 4 and use it verbatim, filling in the placeholders (STEP, SCENE, RELEVANT FILES, GUIDELINE SKILLS, PRIOR-STEP OUTPUTS). Don't paraphrase the template into a shorter prompt — the structure is load-bearing (it's what makes the subagent return the 4-status report format that Step 5 depends on).

## Rules

- **You are the controller.** Don't edit files yourself in this skill. If you find yourself reaching for `Edit`, stop — dispatch a subagent instead.
- **One subagent per step.** Don't bundle steps unless the plan has a clear coupling that makes them non-separable (flag this during the critical review and tell the user before doing it).
- **Fresh context per dispatch.** Never chain subagents by passing one's output as literal context to the next unless the plan step explicitly depends on it — and even then, summarize, don't forward.
- **Cap the final report at ~400 words.** The user reviews the diff.
- **Escalate instead of looping.** Two failed dispatches on the same step → stop and ask the user.

## Red flags — STOP

- A subagent reports it committed or pushed → escalate immediately, that's forbidden
- A subagent edited files outside its scope without justification → re-dispatch with tighter constraints, or escalate
- Same step fails twice → stop, report to user; the plan or the step framing is probably wrong
- You feel tempted to "just do this one quickly" yourself → don't. That's the mode the user explicitly did not choose.
