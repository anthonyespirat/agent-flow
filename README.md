<img width="1573" height="558" alt="image" src="https://github.com/user-attachments/assets/fc31e893-b4f7-4ca0-b666-74e863b984d3" />


# agent-flow

> ⚠️ **Alpha.** APIs, skill names, and the plan format may change without notice. Use on throwaway branches until it stabilizes.

A skill-driven dev workflow for Claude Code (and opencode): **describe → plan → pick mode → execute → test**. No orchestrator. Skills trigger on intent and hand off to each other.

*Built by developers, for developers.*

## Design principles

- **Skills, not orchestration.** Each skill knows what comes next. No top-level router.
- **Plan on disk, not in context.** Survives compaction; reviewable in your editor.
- **Choice of isolation.** In-session for feedback fidelity, subagent-driven for big plans — you pick per task.
- **One todo, owned by the executor.** No double-tracking.
- **Shared debugger.** Any executor loads it when stuck.
- **No destructive ops without explicit ask.** No commits, pushes, or PRs from skills or subagents.
- **Short reports.** Agent output capped (~200–500 words). Raw dumps kill context.
- **[GitNexus](https://github.com/abhigyanpatwari/GitNexus) over grep.** Code exploration goes principaly through the GitNexus knowledge graph — faster, more precise, and far cheaper in tokens than repeated file scans.

## Flow

```mermaid
flowchart LR
    User([💬 describe task]) ==> Plan[📝 <b>plan</b><br/><i>writing-plans</i>]
    Plan ==> File[(📄 .claude/plan/*.md)]
    File ==> Mode{{🎛️ mode?}}
    Mode ==>|in-session| Exec[⚡ <b>execute</b><br/><i>executing-plans</i>]
    Mode ==>|subagents| Sub[🧩 <b>delegate</b><br/><i>subagent-execution</i>]
    Exec ==> Done([✅ test &amp; done])
    Sub ==> Done

    Plan -.-> Ctx[[🔎 ticket + repo context]]
    Exec -.stuck.-> Dbg[[🐞 debugger]]
    Sub -.stuck.-> Dbg

    classDef main fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#111
    classDef side fill:#f3e8ff,stroke:#9333ea,stroke-dasharray:3 3,color:#111
    classDef pick fill:#fde68a,stroke:#d97706,color:#111
    classDef file fill:#e0f2fe,stroke:#0284c7,color:#111

    class Plan,Exec,Sub main
    class Ctx,Dbg side
    class Mode pick
    class File file
```

One path, two execution flavors. Context gathering and the debugger show up only when needed.

## Components

### Skills

| Skill | Role |
|---|---|
| `using-agent-flow` | Entry gate — routes dev tasks into the flow. |
| `writing-plans` | Gathers context, writes `.claude/plan/{slug}.md`. |
| `executing-plans` | Runs the plan in-session. |
| `subagent-execution` | Runs the plan via one subagent per STEP. |
| `debugger` | Diagnostic playbook, loaded on failure. |

### Sub-agents

| Agent | Role |
|---|---|
| `ticket-fetcher` | Pulls a Linear ticket summary. |
| `codebase-explorer` | Maps relevant files via GitNexus. |
| `test-runner` | Typecheck + optional browser check. |

## Plan artifact

A single markdown file at `.claude/plan/{slug}.md`.

- `{slug}` = lowercased Linear ticket id (e.g. `eng-123`) if present, else a short kebab-case slug (≤ 40 chars, no dates).
- Sections: `GOAL`, `APPROACH`, `SKILLS TO APPLY`, `FILES TO CHANGE`, `STEPS` (checklist), `TESTS TO UPDATE/ADD`, `RISKS`, `OUT OF SCOPE`.
- Every STEP starts with `- [ ]` — the executor builds its todo from these.
- On change requests, `writing-plans` overwrites the same file. No `v2`.

## How skills trigger

There is no orchestrator. The skills chain themselves:

1. You describe a dev task → Claude Code's skill matcher fires `using-agent-flow` (meta) which checks whether `writing-plans` applies. For any non-trivial code task, it does.
2. `writing-plans` runs → writes the plan → prints the handoff block.
3. You pick `1` or `2` → that skill fires and executes.
4. Executor ends by offering `test-runner`.

You can also invoke any skill directly:

```
/write-plan add rate limiter to login endpoint
/execute-plan .claude/plan/eng-482.md
```

## Prerequisites

> These must be set up manually. The plugin installer does **not** handle them.

- **Linear MCP** — for `ticket-fetcher` (optional, only if you use Linear refs)
- **GitNexus MCP + indexed repo** — `codebase-explorer` requires it; run `npx gitnexus analyze` in the project first
- **chrome-devtools skill** — used by `test-runner` and referenced by `debugger` for frontend checks
- **TypeScript project** — `test-runner` defaults to `tsc --noEmit`

## Install

### As a Claude Code plugin (recommended)

The repo ships its own single-repo marketplace (`.claude-plugin/marketplace.json`), so two commands install it:

```bash
claude plugin marketplace add anthonyespirat/agent-flow
claude plugin install agent-flow@agent-flow-marketplace
```

Or from inside a Claude Code session:

```
/plugin marketplace add anthonyespirat/agent-flow
/plugin install agent-flow@agent-flow-marketplace
```

Uninstall:

```bash
claude plugin uninstall agent-flow
```

### Manual copy (no plugin system)

Claude Code:

```bash
cp -r skills/* ~/.claude/skills/
cp agents/*.md ~/.claude/agents/
```

opencode:

```bash
cp -r skills/* ~/.config/opencode/skill/
# opencode agents live in a different layout — adapt as needed
```

## Usage

Just describe the task — `using-agent-flow` + `writing-plans` trigger automatically:

```
add a rate limiter to the login endpoint
```

or with a ticket:

```
ENG-482
```

The flow: plan is written, you're shown a summary + the path + two mode options. Reply `1` or `2`.

## Repository layout

```
agent-flow/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── README.md
├── skills/
│   ├── using-agent-flow/SKILL.md
│   ├── writing-plans/SKILL.md
│   ├── executing-plans/SKILL.md
│   ├── subagent-execution/SKILL.md
│   └── debugger/
│       ├── SKILL.md
│       └── references/
│           ├── typecheck.md
│           ├── lint.md
│           ├── lsp.md
│           ├── runtime-errors.md
│           ├── console-logging.md
│           └── chrome-mcp.md
└── agents/
    ├── ticket-fetcher.md
    ├── codebase-explorer.md
    └── test-runner.md
```
