# agent-flow

Structured dev workflow for Claude Code (and opencode): **describe → plan → human-validate → implement → test → human-decide**. Heavy work is delegated to specialized sub-agents so the main conversation stays light.

## Why

Ad-hoc agentic coding tends to skip planning, over-scope, and silently push code. `agent-flow` adds:
- A **plan file on disk** (`.claude/plan/{slug}.md`) as the single source of truth between planning and execution
- **Hard human checkpoints** at plan validation, code review, and decision points
- **Specialized sub-agents** with narrow tool sets and output contracts
- A **shared `debugger` skill** implementer and test-runner call when they get stuck — instead of guessing or flailing

## Flow

```mermaid
flowchart TD
    Start([User: free-form description]) --> Intake[1. Intake<br/>orchestrator<br/>detect ticket + scope]

    Intake -->|Linear ref present| Fetch[2. ticket-fetcher<br/>Linear MCP]
    Intake -->|no ref| Explore
    Fetch --> Explore[3. codebase-explorer<br/>GitNexus MCP<br/>scan skills/]

    Explore --> Plan[4. planner<br/>writes .claude/plan/slug.md]

    Plan --> CP1{🛑 HUMAN<br/>validate plan?}
    CP1 -->|change request| Plan
    CP1 -->|cancel| End1([abort])
    CP1 -->|go| Impl

    Impl[5. implementer<br/>reads plan.md<br/>TodoWrite from STEPS<br/>execute in order]

    Impl -.stuck on error.-> Dbg[debugger skill<br/>references/*.md]
    Dbg -.FIX SUGGESTION.-> Impl
    Dbg -.frontend runtime.-> ChromeDT[chrome-devtools skill]
    ChromeDT -.-> Dbg

    Impl --> CP2{🛑 HUMAN<br/>review diff?}
    CP2 -->|changes| Impl
    CP2 -->|go| Test

    Test[6. test-runner<br/>typecheck + delegates<br/>browser to chrome-devtools]

    Test -.unclear failure.-> Dbg2[debugger skill]
    Dbg2 -.diagnosis.-> Test

    Test -->|pass| CP3
    Test -->|fail| FixDec{ask user:<br/>fix?}
    FixDec -->|yes, iter < 2| Impl
    FixDec -->|no / iter ≥ 2| CP3

    CP3{🛑 HUMAN<br/>next?}
    CP3 -->|PR| PR([open PR])
    CP3 -->|commit| Commit([commit])
    CP3 -->|other| End2([stop])

    classDef human fill:#fde68a,stroke:#d97706,color:#111
    classDef agent fill:#dbeafe,stroke:#2563eb,color:#111
    classDef skill fill:#dcfce7,stroke:#16a34a,color:#111
    classDef artifact fill:#f3e8ff,stroke:#9333ea,color:#111

    class CP1,CP2,CP3,FixDec human
    class Fetch,Explore,Plan,Impl,Test agent
    class Dbg,Dbg2,ChromeDT skill
    class Plan artifact
```

Legend: 🟡 human checkpoints · 🔵 sub-agents · 🟢 skills · ⚪ start/end

## Components

### Orchestrator skill

| Skill | Role |
|---|---|
| `skills/dev-flow/SKILL.md` | Main entrypoint. Routes each step to the right sub-agent and enforces checkpoints. |

### Sub-agents

| Agent | Role | Tools |
|---|---|---|
| `agents/ticket-fetcher.md` | Fetch a Linear ticket, return a short summary | `linear` |
| `agents/codebase-explorer.md` | Map relevant symbols/files via GitNexus, detect guideline skills | `gitnexus`, `Read` |
| `agents/planner.md` | Write `.claude/plan/{slug}.md` with GOAL/APPROACH/STEPS/RISKS | `Read`, `Write` |
| `agents/implementer.md` | Read plan, build todo via `TodoWrite`, execute STEPS in order | `Read`, `Edit`, `Write`, `Glob`, `Grep`, `TodoWrite`, `Bash`, `LSP` |
| `agents/test-runner.md` | Typecheck + delegate browser checks to `chrome-devtools` skill | `Bash`, `Read`, `chrome-devtools` |

### Shared skill

| Skill | Purpose |
|---|---|
| `skills/debugger/SKILL.md` | Diagnostic methodology + routing by symptom |
| `skills/debugger/references/typecheck.md` | TS compile errors (TS2322, TS2339, TS7006, …) |
| `skills/debugger/references/lint.md` | ESLint / Prettier |
| `skills/debugger/references/lsp.md` | `hover`, `goToDefinition`, `findReferences`, call hierarchy |
| `skills/debugger/references/runtime-errors.md` | Stack traces, reproduction, promise rejections |
| `skills/debugger/references/console-logging.md` | Diagnostic instrumentation (with removal discipline) |
| `skills/debugger/references/chrome-mcp.md` | Pointer to the existing `chrome-devtools` skill |

## Plan artifact

The plan is a single markdown file at `.claude/plan/{slug}.md` in the project's working directory.

- `{slug}` = Linear ticket id (lowercased, e.g. `eng-123`) if present, otherwise a short kebab-case slug.
- Format: `GOAL`, `APPROACH`, `SKILLS TO APPLY`, `FILES TO CHANGE`, `STEPS` (checklist), `TESTS TO UPDATE/ADD`, `RISKS`, `OUT OF SCOPE`.
- On change requests, the planner **overwrites** the same file — no stale versions.
- The implementer builds its todo from `STEPS` at the start of execution.

## Install

### As a Claude Code plugin (recommended)

Local path install (for testing before publishing):

```bash
claude plugin install ~/code/agent-flow
```

From a GitHub repo (once pushed):

```bash
claude plugin install anthonyespirat/agent-flow
```

Uninstall:

```bash
claude plugin uninstall agent-flow
```

### As a marketplace entry

Create a separate repo (e.g. `agent-flow-marketplace`) with `.claude-plugin/marketplace.json`:

```json
{
  "name": "your-marketplace",
  "owner": { "name": "Your Team", "email": "team@example.com" },
  "plugins": [
    {
      "name": "agent-flow",
      "source": { "source": "github", "repo": "anthonyespirat/agent-flow" },
      "description": "Structured dev workflow orchestrator",
      "version": "0.1.0"
    }
  ]
}
```

Then users add it:

```bash
claude plugin marketplace add your-org/agent-flow-marketplace
claude plugin install agent-flow@your-marketplace
```

### Manual copy (no plugin system)

Claude Code:

```bash
cp -r skills/dev-flow ~/.claude/skills/
cp -r skills/debugger ~/.claude/skills/
cp agents/*.md ~/.claude/agents/
```

opencode:

```bash
cp -r skills/dev-flow ~/.config/opencode/skill/
cp -r skills/debugger ~/.config/opencode/skill/
# opencode agents live in a different layout — adapt as needed
```

## Prerequisites

- **Linear MCP** — for `ticket-fetcher` (optional, only if you use Linear refs)
- **GitNexus MCP + indexed repo** — `codebase-explorer` requires it; run `npx gitnexus analyze` in the project first
- **chrome-devtools skill** — used by `test-runner` and referenced by `debugger` for frontend checks
- **TypeScript project** — `test-runner` defaults to `tsc --noEmit`

## Usage

Trigger from the main chat:

```
Use dev-flow: add a rate limiter to the login endpoint
```

or, with a ticket:

```
Use dev-flow: ENG-482
```

The orchestrator will announce the steps it will take and pause at each human checkpoint.

## Design principles

- **The orchestrator routes, sub-agents execute.** The main conversation never does raw exploration or editing itself.
- **Plan on disk, not in context.** The plan survives context compaction and can be reviewed by the human in their editor.
- **One todo, owned by the implementer.** Avoid double task tracking between orchestrator and worker.
- **Debugger is a shared skill, not an agent.** Any executor (implementer, test-runner) can load its methodology when stuck, without spawning a new process.
- **No destructive ops without explicit go.** Commit, push, and PR are after CP3, never before.
- **Short reports.** Every sub-agent caps output (200-500 words). Raw dumps kill the orchestrator's context.

## Repository layout

```
agent-flow/
├── .claude-plugin/
│   └── plugin.json
├── README.md
├── skills/
│   ├── dev-flow/
│   │   └── SKILL.md
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
    ├── planner.md
    ├── implementer.md
    └── test-runner.md
```
