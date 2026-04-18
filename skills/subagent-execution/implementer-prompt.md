# Implementer subagent prompt template

Use this template when dispatching a subagent for a single STEP of the plan. Fill in the placeholders, then pass the whole block as the `prompt` argument to the `Agent` tool (with `subagent_type: general-purpose`).

**Do NOT paste the plan file path and let the subagent read it.** Paste the step text, relevant files, skills, and any prior-step context directly. If a type, signature, or config shape matters, paste it inline under `EXCERPTS:` in the dispatch — the template tells the subagent to prefer those over re-reading.

---

```text
You are implementing a single STEP of a plan. You were dispatched fresh —
you do not inherit any prior conversation. Do not read the plan file;
everything you need is below. **Use the EXCERPTS below if present — don't
Read a file just to confirm a type or shape already pasted here.** Read
only when the prompt truly lacks what you need.

STEP: <the - [ ] step text, verbatim>

SCENE: <1-2 sentences — purpose, what came before, what comes after>

RELEVANT FILES (from the plan's FILES TO CHANGE):
- <path1> — <why this step touches it>
- <path2> — <why>

GUIDELINE SKILLS TO APPLY:
- <skill-name>: <1-2 line condensation of the specific rules that apply>
- (or "none — follow conventions observed in the files above")

PRIOR-STEP OUTPUTS (omit if the step is independent of earlier work):
- <file created/modified by step N> — <what it now contains or exports>

EXCERPTS (optional — paste signatures, snippets, configs verbatim):
<paste here, or omit the section>

## Your job

1. If anything is unclear, ask before starting. Don't invent assumptions.
2. Read any file before editing it (Read → Edit).
3. Implement exactly what the STEP specifies. Nothing more.
4. Only edit files in RELEVANT FILES unless unavoidable (justify in NOTES).
5. Follow the GUIDELINE SKILLS as constraints, not suggestions.
6. Self-review your diff, then report.

If stuck — an unexplained typecheck/runtime error, a symbol whose real
shape you need to confirm, or you catch yourself about to guess — invoke
the `debugger` skill via the `Skill` tool. Fold its fix into this STEP;
do NOT expand scope from its diagnosis.

## Forbidden

- No scope expansion beyond this STEP; no plan re-reads.
- No commit/push/PR; no running tests (a separate `test-runner` handles that).

## Report format (return EXACTLY this, ≤150 words; omit any section that doesn't apply)

STATUS: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT

FILES CHANGED:
- <path> — <1 line>

SKILLS APPLIED:
- <skill-name>: <how it shaped the edit> (or "none listed")

NOTES:
- <assumptions, edge cases punted, scope justifications> (or "clean")

IF STATUS is BLOCKED or NEEDS_CONTEXT (otherwise omit this section entirely):
- WHAT STOPPED YOU: <specific>
- WHAT YOU TRIED: <if anything>
- WHAT WOULD UNBLOCK: <more context / different model / split the step>
```

---

## When the controller should deviate from the template

Only two reasons to adapt the template:

- **Skip `PRIOR-STEP OUTPUTS`** when the current step is fully independent of earlier work.
- **Compress `SCENE`** to one sentence when the step is mechanical (e.g. "add this export to this file").

Everything else stays verbatim. The 4 status strings (`DONE`, `DONE_WITH_CONCERNS`, `BLOCKED`, `NEEDS_CONTEXT`) are all the subagent needs to know about status; the controller's handling narrative lives in `SKILL.md` Step 5.
