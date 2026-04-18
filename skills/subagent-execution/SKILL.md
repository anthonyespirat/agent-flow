---
name: subagent-execution
description: Use when a plan file at .claude/plan/{slug}.md exists and the user has chosen subagent-driven execution (mode [2] from writing-plans). Reads the plan, builds a todo from its STEPS, then dispatches one fresh general-purpose subagent per STEP with precisely scoped context. The main conversation stays light — you are the controller, not the implementer.
---

# Subagent-driven execution

Execute a plan by dispatching a fresh subagent per STEP. You (the controller) own the todo and the review; the subagents own the edits. This keeps this conversation light and prevents context pollution across steps.

**Announce at start:** "Using `subagent-execution` to implement `<plan path>` — one subagent per step."

## Why subagents

Each subagent gets exactly the context it needs for its step — no more, no less. They don't inherit this conversation's history. You curate their inputs and review their summaries. Your context stays focused on orchestration.

## Model selection

Set `model` per `Agent` dispatch. **Default to `sonnet`.** Upgrade to `opus` only when the step has an explicit upgrade signal — opus is not the fallback for "I'm not sure". Unset defaults silently become expensive; be deliberate.

- **`haiku`** — mechanical, fully-specified edit in 1–2 files (rename, add a field, isolated pure function); also the right model for `[mechanical]`-tagged steps on the rare occasions you dispatch one instead of executing directly.
- **`sonnet`** — *default*: known-shape multi-file edit, integrating existing patterns, debugging.
- **`opus`** — upgrade only: design judgment, cross-cutting refactor, unfamiliar code, or `BLOCKED` re-dispatch on a reasoning gap.

Upgrade signals (sonnet → opus): 3+ files crossing a module boundary, step says "decide/design/choose", architectural judgment required, or a prior dispatch on the same step failed for reasoning. Downgrade signals (sonnet → haiku): exact edit specified, prior similar step succeeded on `haiku`, step tagged `[mechanical]`. Reviewer subagents match or exceed the implementer's model.

## Mechanical steps — skip dispatch

If a STEP is fully specified with zero judgment calls — running a one-shot shell recipe, scaffolding boilerplate from a known template, generating deterministic config — the controller MAY execute it directly with its own tools instead of dispatching a subagent. A dispatch round-trip costs more than the edit.

Indicators the step is mechanical:

- Tagged `[mechanical]` in the plan (see `writing-plans`).
- Entire content is a shell command, dependency install, or file-copy recipe.
- Zero branches / alternative outcomes; no files need reading beyond the single target.

When you skip dispatch, note it in the final report (`DIRECT EXECUTIONS:` line alongside `SUBAGENT DISPATCHES:`). When unsure, dispatch — this is an exception to "one subagent per step", not a second default path.

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

Group steps into **parallel batches** before dispatching anything.

**Two steps share a batch iff both hold:**
- **Disjoint files.** Their RELEVANT FILES (from Step 2) don't overlap. Treat directory specs (`src/foo/**`) as covering the whole subtree. If a step's file set is vague or missing, it is its own batch.
- **No dependency.** Neither step's PRIOR-STEP OUTPUTS names the other; neither references a symbol/file the other creates, renames, or deletes; neither needs to observe the other's effect to make a decision.

Walk steps top-to-bottom. Open a new batch the moment the next step overlaps files with — or depends on — anyone already in the current batch. **When in doubt, sequentialize.** A wasted-parallel batch costs nothing; a conflicting one costs a re-dispatch and risks silent corruption (two subagents racing on the same file, last write wins).

**For each batch, in order:**

1. `TodoWrite` → mark every step in the batch `in_progress`.
2. **In a single assistant message**, emit one `Agent` tool call per step in the batch (`subagent_type: general-purpose`, `model` chosen per [Model selection](#model-selection), prompt from `./implementer-prompt.md` filled with that step's STEP, SCENE, RELEVANT FILES, GUIDELINE SKILLS, PRIOR-STEP OUTPUTS). One message with N tool calls is the only way they actually run concurrently — separate messages serialize them.
3. Wait for **all** reports in the batch before reacting.
4. Handle each step's status independently per Step 5. A BLOCKED step does **not** invalidate its batch-mates' work — mark successful siblings `completed`, then resolve the failed step before opening the next batch.
5. If any BLOCKED report reveals a plan-level error (contradiction, missing dependency), STOP before opening the next batch.

A batch of one degenerates to the sequential case.



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

DIRECT EXECUTIONS: <count of mechanical steps executed without dispatch>

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
- **One subagent per step** — with one exception: `[mechanical]` / fully-specified steps that the controller executes directly (see [Mechanical steps](#mechanical-steps--skip-dispatch)). Don't bundle unrelated steps.
- **Fresh context per dispatch.** Never chain subagents by passing one's output as literal context to the next unless the plan step explicitly depends on it — and even then, summarize, don't forward.
- **Cap the final report at ~400 words.** The user reviews the diff.
- **Escalate instead of looping.** Two failed dispatches on the same step → stop and ask the user.

## Red flags — STOP

- A subagent reports it committed or pushed → escalate immediately, that's forbidden
- A subagent edited files outside its scope without justification → re-dispatch with tighter constraints, or escalate
- Same step fails twice → stop, report to user; the plan or the step framing is probably wrong
- You feel tempted to "just do this one quickly" yourself → don't. That's the mode the user explicitly did not choose.
