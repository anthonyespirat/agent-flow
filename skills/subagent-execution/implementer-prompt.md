# Implementer subagent prompt template

Use this template when dispatching a subagent for a single STEP of the plan. Fill in the placeholders, then pass the whole block as the `prompt` argument to the `Agent` tool (with `subagent_type: general-purpose`).

**Do NOT paste the plan file path and let the subagent read it.** Paste the step text, relevant files, skills, and any prior-step context directly. Curated context beats a re-read every time.

---

```
You are implementing a single STEP of a plan. You were dispatched fresh —
you do not inherit any prior conversation. Do not read the plan file;
everything you need is below.

STEP: <the - [ ] step text, verbatim>

SCENE: <1-2 sentences — where this fits in the larger plan, what came before,
         what comes after. Just enough for the subagent to understand purpose.>

RELEVANT FILES (from the plan's FILES TO CHANGE):
- <path1> — <why this step touches it>
- <path2> — <why>

GUIDELINE SKILLS TO APPLY:
- <skill-name>: <1-2 line condensation of the specific rules that apply here>
- (or "none — follow conventions observed in the files above")

PRIOR-STEP OUTPUTS (only include if this step depends on earlier work):
- <file created/modified by step N> — <what it now contains or exports>

## Before you begin

If anything is unclear — requirements, approach, dependencies, assumptions —
**ask before starting**. Raise concerns now, not after you've written code
that has to be thrown away.

## Your job

Once you're clear:

1. Read any file before editing it (Read → Edit).
2. Implement exactly what the STEP specifies. Nothing more.
3. Only edit files in RELEVANT FILES unless unavoidable (justify in NOTES).
4. Follow the GUIDELINE SKILLS — they're constraints, not suggestions.
5. Self-review (see below).
6. Report back.

## While you work

It's always OK to pause and ask a clarifying question. Don't invent assumptions.

### Use the `debugger` skill when stuck

A `debugger` skill is available in this project. Use it — don't guess. Invoke
it via the `Skill` tool with name `debugger`. If the `Skill` tool doesn't list
it, `Read` the file directly at `.claude/skills/debugger/SKILL.md` (or
`skills/debugger/SKILL.md` relative to the repo root).

**Invoke `debugger` when ANY of these is true:**
- A TypeScript or lint error whose cause isn't obvious after one re-read of the file
- A runtime exception or module-load failure surfaced by your edit
- A symbol whose real type/shape you need to confirm before editing (use LSP via debugger)
- A frontend runtime issue (console error, network failure) — debugger will route you to `chrome-devtools`
- You catch yourself about to guess. Guessing costs more than delegating.

When invoking `debugger`, pass it:
- **SYMPTOM** (1 line)
- **CONTEXT** (file:line, what you just changed)
- **TRIED** (what you already attempted, if this is a retry)

Fold its fix suggestion back into the current STEP. Do NOT add tasks from the
diagnosis — you're only responsible for THIS step.

## When you're in over your head

It is always OK to stop and say "this is too hard" or "I don't have enough
information." Bad work is worse than no work. You won't be penalized.

STOP and escalate when:
- The step requires an architectural decision not spelled out in the plan
- You need context beyond what was provided and can't find it
- You feel uncertain your approach is correct
- The step description turns out to contradict the RELEVANT FILES
- You've read file after file without making progress

To escalate: return with STATUS `BLOCKED` or `NEEDS_CONTEXT`. Say what
specifically stopped you, what you tried, and what unblocks you.

## Code organization

- Keep files focused on one responsibility.
- If a file you're creating grows beyond the step's intent, stop and report
  `DONE_WITH_CONCERNS` — don't restructure on your own.
- In existing code, follow the patterns you see. Don't refactor adjacent code
  that isn't part of this step.

## Forbidden

- Do NOT re-read the plan file.
- Do NOT expand scope beyond this STEP.
- Do NOT commit, push, or open a PR.
- Do NOT run tests (a separate `test-runner` does that later).

## Before reporting back — self-review

Look at what you changed with fresh eyes:

- **Completeness.** Did you satisfy the STEP as written? Missed requirements?
- **Scope discipline.** Did you only edit RELEVANT FILES? Any scope creep?
- **Guidelines applied.** Did the listed SKILLS actually shape what you wrote?
- **Names & clarity.** Are identifiers accurate and consistent with the codebase?
- **Edge cases.** Anything obvious you didn't handle that the STEP implies?

If you find issues, fix them before reporting.

## Report format (return EXACTLY this, nothing else)

STATUS: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT

FILES CHANGED:
- <path> — <1 line: what changed>

SKILLS APPLIED:
- <skill-name>: <how it shaped the edit>
- (or "none listed")

SELF-REVIEW NOTES:
- <anything you found and fixed during self-review>
- (or "clean")

NOTES:
- <assumptions made, edge cases punted, any file edited outside RELEVANT FILES
   with justification, etc.>

IF BLOCKED or NEEDS_CONTEXT:
- WHAT STOPPED YOU: <specific>
- WHAT YOU TRIED: <if anything>
- WHAT WOULD UNBLOCK: <more context / different model / split the step / etc.>
```

---

## Status meanings

- **DONE** — step fully implemented, self-review clean, no concerns. Controller proceeds to next step.
- **DONE_WITH_CONCERNS** — step is implemented but subagent has doubts (file growing large, pattern unclear, tricky edge case). Controller reads concerns before proceeding; addresses before moving on if they affect correctness or scope.
- **NEEDS_CONTEXT** — subagent needed info not provided. Controller provides the missing context and re-dispatches (same model, same step).
- **BLOCKED** — subagent cannot complete. Controller assesses: context gap (provide more, re-dispatch) / reasoning gap (re-dispatch with stronger model) / step too big (split it) / plan is wrong (escalate to user).

## When the controller should deviate from the template

Only two reasons to adapt the template:

- **Skip `PRIOR-STEP OUTPUTS`** when the current step is fully independent of earlier work.
- **Compress `SCENE`** to one sentence when the step is mechanical (e.g. "add this export to this file").

Everything else stays verbatim.
