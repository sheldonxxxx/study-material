# GSD Architecture Analysis

> Research output: Architecture pattern identification, key modules, and component communication flow.

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

## Key Modules and Responsibilities

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

**Key Design Pattern — Compound Init Commands:**
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
   → Returns JSON: project info, config, state, phase details
3. Resolve model: gsd-tools.cjs resolve-model <agent-name>
4. Spawn agent(s): Task(subagent_type="gsd-xxx", prompt="...")
5. Collect results
6. Update state: gsd-tools.cjs state update/patch/advance-plan
```

---

## Communication and Flow Between Components

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

## Entry Point Analysis: How CLI Commands Work

### Architecture: Single CLI with Subcommand Routing

`gsd-tools.cjs` is a single Node.js CLI entry point (~920 lines) that:

1. **Parses global flags** (`--cwd`, `--ws`, `--raw`, `--pick`)
2. **Routes to command handlers** via `runCommand()` switch statement
3. **Delegates to domain modules** (state.cjs, phase.cjs, etc.)

**Command Structure:**
```
gsd-tools <command> [subcommand] [args] [--flags]
```

**Example Commands:**
```bash
# State management
gsd-tools state load
gsd-tools state update phase "Phase 3"
gsd-tools state patch --field value

# Phase operations
gsd-tools phase add "User authentication" --id AUTH
gsd-tools phase complete 3

# Context loading (for workflows)
gsd-tools init execute-phase 3
gsd-tools init plan-phase 3
gsd-tools init new-project

# Validation
gsd-tools verify plan-structure .planning/phases/03/03-01-PLAN.md
gsd-tools verify artifacts .planning/phases/03/03-01-PLAN.md

# Templates
gsd-tools template fill summary --phase 3 --plan 1
```

### Init Commands: Context Payload Generation

`init.cjs` provides compound context loading — each workflow type has a dedicated init function that aggregates all necessary context into a single JSON payload.

**Example: `cmdInitExecutePhase`** returns:
```json
{
  "executor_model": "sonnet",
  "verifier_model": "sonnet",
  "commit_docs": true,
  "parallelization": true,
  "phase_dir": "03-user-auth",
  "phase_number": "03",
  "phase_name": "User Authentication",
  "plans": ["03-01-PLAN.md", "03-02-PLAN.md"],
  "incomplete_plans": ["03-02-PLAN.md"],
  "milestone_version": "v1.0",
  "branch_name": "gsd/phase-03-user-auth"
}
```

### Output Format

All commands output JSON to stdout by default:
```javascript
function output(result, raw, rawValue) {
  const json = JSON.stringify(result, null, 2);
  // Large payloads (>50KB) written to temp file, path returned as @file:
  fs.writeSync(1, data);
}
```

**Flags:**
- `--raw` — Output raw string instead of JSON
- `--pick <field>` — Extract single field from JSON output

---

## Architectural Pattern Summary

### GSD Is NOT a:
- **Traditional CLI tool** — It's a context engineering framework
- **Monolithic agent** — Each agent is specialized, spawned fresh per task
- **Pipeline with fixed stages** — Workflows are composable, with checkpoints and branches

### GSD IS a:
- **Meta-prompting system** — Generates context-rich prompts for AI agents
- **Multi-agent orchestrator** — Coordinates specialized agents with shared state
- **Spec-driven development** — Requirements flow through research → plans → execution → verification
- **File-based state machine** — Persistent `.planning/` directory as single source of truth

### Key Architectural Insights

1. **Fresh Context Per Agent** — Every spawned agent gets a clean context window (up to 200K tokens), eliminating context rot

2. **Thin Orchestrators** — Workflows load context and spawn agents; they don't execute tasks themselves

3. **File-Based State** — All state in `.planning/` as human-readable Markdown/JSON; survives context resets, inspectable by humans and agents

4. **Absent = Enabled** — Feature flags default to `true` when absent from `config.json`

5. **Defense in Depth** — Plans verified before execution, execution produces atomic commits, post-execution verification, UAT as final gate

6. **Runtime Abstraction** — Single codebase supports Claude Code, OpenCode, Gemini CLI, Codex, Copilot, Antigravity via installer-mediated transformation
