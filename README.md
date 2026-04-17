<img width="1559" height="642" alt="image" src="https://github.com/user-attachments/assets/98384697-11d1-4895-8962-98a37532fe65" />


# forge

> ⚠️ **Alpha.** APIs, skill names, and the plan format may change without notice. Use on throwaway branches until it stabilizes.
>
> Currently biased toward **TypeScript** projects (typecheck defaults, suggested skills, assumed tooling). PRs welcome to make it language-agnostic.

A skill-driven dev workflow for Claude Code (and opencode): **describe → plan → pick mode → execute → test**. No orchestrator. Skills trigger on intent and hand off to each other.

*Built by developers, for developers.*

## Design principles

- **Skills, not orchestration.** Each skill knows what comes next. No top-level router.
- **Plan on disk, not in context.** Survives compaction; reviewable in your editor.
- **Choice of isolation.** In-session for feedback fidelity, subagent-driven for big plans — you pick per task.
- **Shared debugger.** Any executor loads it when stuck.
- **No destructive ops without explicit ask.** No commits, pushes, or PRs from skills or subagents.
- **Short reports.** Agent output capped (~200–500 words). Raw dumps kill context.
- **🚀 [GitNexus](https://github.com/abhigyanpatwari/GitNexus) over grep.** Code exploration goes principaly through the GitNexus knowledge graph — faster, more precise, and far cheaper in tokens than repeated file scans.
- **🦴 [Caveman](https://github.com/JuliusBrussee/caveman) for agent-to-agent.** Selected skills and sub-agents emit their machine-consumed sections (reports handed to another skill, not shown to the user) in caveman style — cuts the payload roughly in half while preserving signal.

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
| `using-forge` | Entry gate — routes dev tasks into the flow. |
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

1. You describe a dev task → Claude Code's skill matcher fires `using-forge` (meta) which checks whether `writing-plans` applies. For any non-trivial code task, it does.
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
- **[caveman](https://github.com/JuliusBrussee/caveman) skill** — agent-to-agent payloads (e.g. `codebase-explorer` → `writing-plans`) compress through it
- **chrome-devtools skill** — used by `test-runner` and referenced by `debugger` for frontend checks
- **TypeScript project** — `test-runner` defaults to `tsc --noEmit`

## Install

### As a Claude Code plugin (recommended)

The repo ships its own single-repo marketplace (`.claude-plugin/marketplace.json`), so two commands install it:

```bash
claude plugin marketplace add anthonyespirat/forge
claude plugin install forge@forge-marketplace
```

Or from inside a Claude Code session:

```
/plugin marketplace add anthonyespirat/forge
/plugin install forge@forge-marketplace
```

Uninstall:

```bash
claude plugin uninstall forge
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

Just describe the task — `using-forge` + `writing-plans` trigger automatically:

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
forge/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── README.md
├── skills/
│   ├── using-forge/SKILL.md
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

## Inspiration

- [superpowers](https://github.com/obra/superpowers) — the original skill-driven workflow this project riffs on.
