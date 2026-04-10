# GSD Architecture

> High-level architectural pattern, module responsibilities, and command flow.

## Architectural Pattern: Multi-Agent Orchestration with Thin Coordinators

GSD implements a **meta-prompting framework** — a layered architecture where lightweight orchestrators coordinate specialized agents, each operating with a fresh context window.

```
┌─────────────────────────────────────────────────────────────┐
│                         USER                                 │
│               /gsd:command [args]                            │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                  COMMAND LAYER                                │
│  commands/gsd/*.md — Prompt-based slash commands            │
│  (Claude Code: /gsd:cmd, OpenCode: /gsd-cmd, Codex: $gsd-cmd) │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                  WORKFLOW LAYER                               │
│  get-shit-done/workflows/*.md — Orchestration logic          │
│  Thin coordinators: load context, spawn agents, route state │
└───────┬─────────────┬──────────────────┬────────────────────┘
        │             │                  │
┌───────▼──────┐ ┌────▼─────┐ ┌──────────▼──────────┐
│   AGENT      │ │  AGENT   │ │      AGENT          │
│  (specialized)│ │(specialized)│ │   (specialized)  │
│  gsd-executor │ │gsd-planner │ │  gsd-researcher  │
└───────┬──────┘ └────┬─────┘ └──────────┬──────────┘
        │             │                  │
┌───────▼──────────────────────────────────────────────┐
│                 CLI TOOLS LAYER                        │
│  get-shit-done/bin/gsd-tools.cjs                      │
│  State, config, phase, roadmap, verify, templates      │
└──────────────────────────┬───────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────┐
│              FILE SYSTEM (.planning/)                   │
│  PROJECT.md | STATE.md | ROADMAP.md | config.json     │
│  phases/ | requirements/ | todos/                      │
└───────────────────────────────────────────────────────┘
```

---

## Major Modules and Responsibilities

### 1. CLI Tools Layer (`bin/gsd-tools.cjs`)

**Role:** Central nervous system — single entry point for all tooling operations.

**Entry Point Pattern:**
```bash
node gsd-tools.cjs <command> [args] [--raw] [--pick <field>] [--cwd <path>]
```

**Core Commands (via `runCommand` router):**

| Module | File | Responsibility |
|--------|------|----------------|
| **core** | `lib/core.cjs` | Error handling, output formatting, path helpers, git utils, planning dir resolution |
| **state** | `lib/state.cjs` | STATE.md CRUD, progression engine, metrics recording |
| **phase** | `lib/phase.cjs` | Phase directory ops, decimal numbering, plan indexing |
| **roadmap** | `lib/roadmap.cjs` | ROADMAP.md parsing, phase extraction |
| **config** | `lib/config.cjs` | config.json read/write, section initialization |
| **verify** | `lib/verify.cjs` | Plan structure validation, reference checking, commit verification |
| **template** | `lib/template.cjs` | Template selection and filling |
| **frontmatter** | `lib/frontmatter.cjs` | YAML frontmatter CRUD |
| **init** | `lib/init.cjs` | Compound context loading per workflow type |
| **milestone** | `lib/milestone.cjs` | Milestone archival, requirements marking |
| **commands** | `lib/commands.cjs` | Misc utilities (slug, timestamp, todos, scaffolding) |
| **model-profiles** | `lib/model-profiles.cjs` | Agent-to-model mapping per profile tier |
| **security** | `lib/security.cjs` | Path traversal prevention, prompt injection detection |
| **uat** | `lib/uat.cjs` | UAT parsing, verification debt tracking |
| **profile-pipeline** | `lib/profile-pipeline.cjs` | Session scanning and message extraction |
| **profile-output** | `lib/profile-output.cjs` | Profile generation and questionnaire |
| **workstream** | `lib/workstream.cjs` | Parallel workstream management |

**Key Design Pattern - Compound Init Commands:**
```javascript
// init.cjs exports context-loading functions for each workflow:
// cmdInitExecutePhase(), cmdInitPlanPhase(), cmdInitNewProject(), etc.
// Each returns a JSON payload with all context needed for that workflow.
```

### 2. Agent Layer (`agents/*.md`)

**Role:** Specialized task executors with defined roles, tools, and behaviors.

**16 Agent Types:**

| Category | Agents | Parallelism |
|----------|--------|-------------|
| **Researchers** | gsd-project-researcher, gsd-phase-researcher, gsd-ui-researcher, gsd-advisor-researcher | 4 parallel |
| **Synthesizers** | gsd-research-synthesizer | Sequential |
| **Planners** | gsd-planner, gsd-roadmapper | Sequential |
| **Checkers** | gsd-plan-checker, gsd-integration-checker, gsd-ui-checker, gsd-nyquist-auditor | Sequential (verification loop) |
| **Executors** | gsd-executor | Parallel within waves |
| **Verifiers** | gsd-verifier | Sequential |
| **Mappers** | gsd-codebase-mapper | 4 parallel |
| **Debuggers** | gsd-debugger | Sequential (interactive) |
| **Auditors** | gsd-ui-auditor | Sequential |

**Agent Frontmatter Schema:**
```yaml
---
name: gsd-executor
description: Role and purpose
tools: Read, Write, Edit, Bash, Grep, Glob  # Allowed tools
permissionMode: acceptedits
color: yellow
---
```

### 3. Workflow Layer (`workflows/*.md`)

**Role:** Orchestration logic — thin coordinators that never do heavy lifting.

**46 Workflows** including:
- `execute-phase.md` — Wave-based parallel execution orchestrator
- `execute-plan.md` — Per-plan execution with atomic commits
- `plan-phase.md` — Planning orchestration with checker loop
- `discuss-phase.md` — Context gathering with user
- `new-project.md` — Project initialization pipeline
- `verify-work.md` — Post-execution verification

**Workflow Execution Pattern:**
```
1. Parse arguments
2. Load context: gsd-tools.cjs init <workflow> <phase>
   -> Returns JSON: project info, config, state, phase details
3. Resolve model: gsd-tools.cjs resolve-model <agent-name>
4. Spawn agent(s): Task(subagent_type="gsd-xxx", prompt="...")
5. Collect results
6. Update state: gsd-tools.cjs state update/patch/advance-plan
```

---

## How Commands Flow Through the System

### Entry Point: CLI Command Invocation

**Flow: User Slash Command to Agent Execution**

```
1. User types: /gsd:execute-phase 3
   ↓
2. Claude Code loads: commands/gsd/execute-phase.md
   ↓
3. Workflow prompt executes in current Claude session
   ↓
4. Workflow loads context:
   INIT=$(node gsd-tools.cjs init execute-phase "3")
   ↓
5. gsd-tools.cjs parses command, routes to init.cjs
   ↓
6. init.cjs loads config.json, STATE.md, finds phase dir
   ↓
7. Returns JSON payload to workflow
   ↓
8. Workflow spawns executor agents per plan:
   Task(subagent_type="gsd-executor", model="sonnet", prompt="...")
   ↓
9. Executor agent loads plan, executes tasks, creates commits
   ↓
10. Executor creates SUMMARY.md
   ↓
11. Orchestrator updates STATE.md, ROADMAP.md
```

### State Propagation Pattern

```
PROJECT.md ────────────────────────────────────────────► All agents
REQUIREMENTS.md ───────────────────────────────────────► Planner, Verifier
ROADMAP.md ────────────────────────────────────────────► Orchestrators
STATE.md ───────────────────────────────────────────────► All agents
CONTEXT.md (per phase) ────────────────────────────────► Researcher, Planner, Executor
RESEARCH.md (per phase) ───────────────────────────────► Planner, Plan Checker
PLAN.md (per plan) ────────────────────────────────────► Executor
SUMMARY.md (per plan) ─────────────────────────────────► Verifier
UI-SPEC.md (per phase) ────────────────────────────────► Executor, UI Auditor
```

### Wave-Based Parallel Execution

During `execute-phase`, plans are analyzed for dependencies and grouped into waves:

```
Wave Analysis:
  Plan 01 (no deps)      ─┐
  Plan 02 (no deps)      ─┤── Wave 1 (parallel)
  Plan 03 (depends: 01)  ─┤── Wave 2 (waits for Wave 1)
  Plan 04 (depends: 02)  ─┘
  Plan 05 (depends: 03,04) ── Wave 3 (waits for Wave 2)
```

**Parallel Commit Safety (two mechanisms):**

1. **`--no-verify` commits** — Parallel agents skip pre-commit hooks to avoid hook contention
2. **STATE.md file locking** — Lockfile-based mutual exclusion prevents read-modify-write races

### Model Resolution Flow

```
config.json: { model_profile: "balanced" }
                        ↓
gsd-tools.cjs resolve-model gsd-executor
                        ↓
model-profiles.cjs lookup:
  gsd-executor: { quality: 'opus', balanced: 'sonnet', budget: 'sonnet' }
                        ↓
Returns: "sonnet" (or full model ID if resolve_model_ids: true)
```

---

## Key Design Decisions with Rationale

### 1. Single CLI Entry Point

`gsd-tools.cjs` is a single Node.js CLI (~920 lines) that routes all commands through `runCommand()` switch.

**Rationale:** Consistent interface, single dependency to install, easy to version. All domain modules imported centrally.

### 2. File-Based State

All project state lives in `.planning/` directory as human-readable Markdown/JSON files.

**Rationale:** Survives context resets, inspectable by both humans and agents, git-tracked, no database needed.

### 3. Compound Init Commands

Each workflow type has a dedicated init function in `init.cjs` that aggregates all context into a single JSON payload.

**Rationale:** Workflows stay thin; each init function knows exactly what that workflow needs; context loading is testable in isolation.

### 4. "Absent = Enabled" Feature Flags

Feature flags in `config.json` default to `true` when absent.

**Rationale:** Reduces configuration overhead; new features enabled by default without explicit opt-in.

### 5. Defense in Depth Verification

Plans verified before execution, execution produces atomic commits, post-execution verification, UAT as final gate.

**Rationale:** Each layer catches different failure modes; no single point of trust.

### 6. Runtime Abstraction

Single codebase supports Claude Code, OpenCode, Gemini CLI, Codex, Copilot via installer-mediated transformation.

**Rationale:** Write once, run everywhere; installer adapts command syntax per runtime.

### 7. Thin Workflows

Workflows only load context, spawn agents, and route results — they never execute tasks themselves.

**Rationale:** Workflows are orchestration logic, not business logic; keeps them reusable across different agent implementations.

---

## Directory Structure

```
get-shit-done/
├── bin/                          # Installation entry point
│   └── install.js                # Main installer (174KB)
├── get-shit-done/                # Core library package
│   ├── bin/
│   │   ├── gsd-tools.cjs         # CLI tools (36KB)
│   │   └── lib/                  # Core modules (19 .cjs files)
│   ├── commands/                 # Command implementations (57 .md)
│   ├── references/               # Reference docs (16 .md)
│   ├── templates/               # Project templates (34 files)
│   └── workflows/               # Execution workflows (57 .md)
├── agents/                      # Agent definitions (17 .md)
├── commands/                     # CLI command definitions (57 .md)
├── hooks/                        # Git hooks
├── scripts/                      # Build and utility scripts
├── tests/                        # Test suite (50 .test.cjs)
└── .planning/                    # Project state (created at init)
```

---

## What GSD Is NOT

- **Traditional CLI tool** — It's a context engineering framework
- **Monolithic agent** — Each agent is specialized, spawned fresh per task
- **Pipeline with fixed stages** — Workflows are composable, with checkpoints and branches

## What GSD IS

- **Meta-prompting system** — Generates context-rich prompts for AI agents
- **Multi-agent orchestrator** — Coordinates specialized agents with shared state
- **Spec-driven development** — Requirements flow through research → plans → execution → verification
- **File-based state machine** — Persistent `.planning/` directory as single source of truth
