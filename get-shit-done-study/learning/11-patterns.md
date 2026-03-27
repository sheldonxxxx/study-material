# Design Patterns in GSD

> Enumerated design patterns with specific file references and code evidence.

## Pattern 1: Thin Orchestrator Pattern

**Description:** Workflows act as lightweight coordinators that load context, spawn agents, and route results — they never execute business logic themselves.

**Evidence:** `workflows/execute-phase.md`
```
1. Parse arguments
2. Load context: gsd-tools.cjs init <workflow> <phase>
3. Resolve model: gsd-tools.cjs resolve-model <agent-name>
4. Spawn agent(s): Task(subagent_type="gsd-xxx", prompt="...")
5. Collect results
6. Update state: gsd-tools.cjs state update/patch/advance-plan
```

**When to Use:** When orchestrating multi-agent workflows where business logic should be delegated to specialized agents.

---

## Pattern 2: File-Based State Pattern

**Description:** All project state persisted as human-readable Markdown/JSON files in `.planning/` directory, enabling survival across context resets and human inspection.

**Evidence:** `get-shit-done/bin/lib/state.cjs`, `get-shit-done/bin/lib/phase.cjs`

State files:
- `STATE.md` — Current position, decisions, blockers
- `ROADMAP.md` — Phase plans and requirements
- `PROJECT.md` — Project metadata and goals
- `config.json` — Feature flags and model profiles
- `phases/XX-name/PLAN.md` — Per-plan execution details
- `phases/XX-name/SUMMARY.md` — Execution results

**When to Use:** When state must survive agent context resets, be git-tracked, and remain human-readable.

---

## Pattern 3: Fresh Context Per Agent Pattern

**Description:** Every spawned agent receives a clean context window (up to 200K tokens), eliminating context contamination and "context rot."

**Evidence:** `workflows/execute-plan.md` spawns Task with fresh subagent:
```
Task(subagent_type="gsd-executor", model="sonnet", prompt="...")
```

**Agent definition:** `agents/gsd-executor.md`
```yaml
---
name: gsd-executor
description: Role and purpose
tools: Read, Write, Edit, Bash, Grep, Glob
permissionMode: acceptedits
color: yellow
---
```

**When to Use:** When agents perform complex, long-running tasks that could suffer from context pollution.

---

## Pattern 4: Compound Init Command Pattern

**Description:** Each workflow type has a dedicated init function that aggregates all necessary context into a single JSON payload, rather than workflows querying multiple sources.

**Evidence:** `get-shit-done/bin/lib/init.cjs`

```javascript
// init.cjs exports context-loading functions for each workflow:
// cmdInitExecutePhase(), cmdInitPlanPhase(), cmdInitNewProject(), etc.
```

Example payload from `cmdInitExecutePhase`:
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
  "incomplete_plans": ["03-02-PLAN.md"]
}
```

**When to Use:** When a workflow needs data from multiple sources (config, state, roadmap, phase dir) and you want to keep workflows thin.

---

## Pattern 5: Wave-Based Parallel Execution Pattern

**Description:** Plans are analyzed for dependencies and grouped into waves, with plans in the same wave executing in parallel while waves execute sequentially.

**Evidence:** `workflows/execute-phase.md`
```
Wave Analysis:
  Plan 01 (no deps)      ─┐
  Plan 02 (no deps)      ─┤── Wave 1 (parallel)
  Plan 03 (depends: 01)  ─┤── Wave 2 (waits for Wave 1)
  Plan 04 (depends: 02)  ─┘
  Plan 05 (depends: 03,04) ── Wave 3 (waits for Wave 2)
```

**When to Use:** When independent plans can execute concurrently to maximize throughput while respecting dependencies.

---

## Pattern 6: Parallel Commit Safety Pattern

**Description:** Two mechanisms ensure safe parallel commits: `--no-verify` skips pre-commit hooks, and STATE.md uses lockfile-based mutual exclusion.

**Evidence:** `get-shit-done/bin/lib/state.cjs` (lockfile), workflows use `--no-verify` for parallel agents.

**When to Use:** When running multiple agents that might commit simultaneously.

---

## Pattern 7: Model Resolution Pattern

**Description:** Agent-to-model mapping defined in `model-profiles.cjs` and resolved at runtime based on `config.json` profile setting.

**Evidence:** `get-shit-done/bin/lib/model-profiles.cjs`

```
config.json: { model_profile: "balanced" }
                        ↓
gsd-tools.cjs resolve-model gsd-executor
                        ↓
model-profiles.cjs lookup:
  gsd-executor: { quality: 'opus', balanced: 'sonnet', budget: 'sonnet' }
                        ↓
Returns: "sonnet"
```

**When to Use:** When different agents require different model capabilities and you want runtime flexibility.

---

## Pattern 8: Spec-Driven Development Pattern

**Description:** Requirements flow through a defined lifecycle: research → context → plans → execution → verification → UAT.

**Evidence:** State propagation across phases:
```
REQUIREMENTS.md ──► Planner, Verifier
CONTEXT.md ───────► Researcher, Planner, Executor
RESEARCH.md ──────► Planner, Plan Checker
PLAN.md ──────────► Executor
SUMMARY.md ───────► Verifier
```

**When to Use:** When building complex systems where traceability from requirements to implementation is critical.

---

## Pattern 9: Defense in Depth Verification Pattern

**Description:** Multiple verification layers at different stages catch different failure modes.

**Evidence:** `workflows/verify-phase.md`, `workflows/verify-work.md`
1. **Pre-execution:** Plan checker validates structure
2. **During execution:** Atomic commits preserve working state
3. **Post-execution:** Verifier confirms artifacts exist
4. **Final gate:** UAT (User Acceptance Testing)

**When to Use:** When failure modes are diverse and you cannot trust a single verification layer.

---

## Pattern 10: Absent = Enabled Pattern

**Description:** Feature flags in `config.json` default to `true` when absent, enabling new features by default.

**Evidence:** `get-shit-done/bin/lib/config.cjs` — missing flags return `true`.

**When to Use:** When you want minimal configuration overhead and new features enabled by default.

---

## Pattern 11: Single CLI Entry Point Pattern

**Description:** All tooling operations routed through a single `gsd-tools.cjs` with subcommand routing to domain modules.

**Evidence:** `get-shit-done/bin/gsd-tools.cjs` (~920 lines)
```
gsd-tools <command> [subcommand] [args] [--flags]

# Examples:
gsd-tools state load
gsd-tools phase add "User auth" --id AUTH
gsd-tools init execute-phase 3
gsd-tools verify plan-structure .planning/phases/03/03-01-PLAN.md
```

**When to Use:** When you want a single dependency that provides all tooling operations.

---

## Pattern 12: Runtime Abstraction Pattern

**Description:** Single codebase adapts to multiple AI runtimes (Claude Code, OpenCode, Gemini CLI, Codex) via installer-mediated transformation.

**Evidence:** `bin/install.js` (174KB) transforms commands per runtime:
- Claude Code: `/gsd:command`
- OpenCode: `/gsd-cmd`
- Codex: `$gsd-cmd`

**When to Use:** When targeting multiple AI coding platforms with a single codebase.

---

## Pattern 13: Milestone-Bounded Execution Pattern

**Description:** Work is organized into milestones containing phases, with milestone completion archiving requirements and advancing version.

**Evidence:** `get-shit-done/bin/lib/milestone.cjs`

```
Milestone v1.0
├── Phase 1: Foundation (plans 01-02)
├── Phase 2: Auth (plans 03-05)
└── Phase 3: API (plans 06-08)

Complete milestone → archive requirements, bump version
```

**When to Use:** When organizing large projects into release-able increments.

---

## Pattern 14: Agent Specialization Pattern

**Description:** Different agents have distinct roles, tools, and parallelism characteristics rather than one general-purpose agent.

**Evidence:** `agents/` directory with 16 agent types:

| Category | Agents | Parallelism |
|----------|--------|-------------|
| Researchers | gsd-project-researcher, gsd-phase-researcher | Parallel |
| Planners | gsd-planner, gsd-roadmapper | Sequential |
| Checkers | gsd-plan-checker, gsd-integration-checker | Sequential |
| Executors | gsd-executor | Parallel within waves |
| Verifiers | gsd-verifier | Sequential |

**When to Use:** When different tasks benefit from different agent capabilities and contexts.

---

## Pattern 15: Frontmatter-Driven Configuration Pattern

**Description:** Plans, agents, and workflows use YAML frontmatter for declarative configuration parsed by `frontmatter.cjs`.

**Evidence:** Plan frontmatter (`03-01-PLAN.md`):
```yaml
---
phase: 03-user-auth
plan: 01
type: execute
wave: 1
depends_on: []
files_modified: [src/models/user.ts, src/api/auth.ts]
autonomous: true
requirements: [AUTH-01, AUTH-02]
---
```

Agent frontmatter (`agents/gsd-executor.md`):
```yaml
---
name: gsd-executor
description: Executes plan tasks
tools: Read, Write, Edit, Bash, Grep, Glob
permissionMode: acceptedits
color: yellow
---
```

**When to Use:** When configuration should be human-readable, git-tracked, and easily parsed without complex schema validation.

---

## Pattern 16: Context Aggregation Pattern

**Description:** Multiple context sources (PROJECT.md, STATE.md, ROADMAP.md, phase-specific files) are aggregated and presented to agents as a unified context block.

**Evidence:** `get-shit-done/bin/lib/init.cjs` `cmdInitExecutePhase()` loads:
- config.json
- STATE.md
- ROADMAP.md
- Phase directory contents
- Plan files

Returns single JSON payload to workflow.

**When to Use:** When agents need a complete picture of project state without hunting through multiple files.

---

## Pattern 17: Atomic Commit per Task Pattern

**Description:** Each discrete unit of work (plan execution) produces an atomic git commit, enabling fine-grained rollback and audit trail.

**Evidence:** `workflows/execute-plan.md`
```
For each plan:
  1. Spawn executor agent
  2. Agent executes tasks
  3. Agent creates atomic commit
  4. Create SUMMARY.md
  5. Update STATE.md
```

**When to Use:** When you need reversible, traceable work units that can be rolled back independently.
