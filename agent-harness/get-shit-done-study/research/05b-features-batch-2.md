# Feature Batch 2: Execution & Verification Features

## Features Covered
- Feature 4: Wave-Based Phase Execution
- Feature 5: Work Verification (UAT)
- Feature 6: Quick Task Mode

---

## Feature 4: Wave-Based Phase Execution

### Feature Name and Description

**Wave-Based Phase Execution** is the orchestration layer that executes phase plans with parallelism, freshness, and traceability. The orchestrator (`execute-phase.md`) stays lean (~10-15% context) and spawns fresh subagents for each plan, each with a full 200k-token context window. Plans are grouped into waves based on dependencies; within each wave, parallel execution occurs when `parallelization=true`. Atomic per-task git commits ensure bisect-friendly history.

### Core Implementation Files

#### `get-shit-done/workflows/execute-phase.md`
The main orchestrator workflow. Coordinates plan discovery, wave grouping, subagent spawning, checkpoint handling, and verification.

#### `agents/gsd-executor.md`
The executor agent definition. Spawned by `execute-phase` to execute individual plans. Handles task execution, deviation rules, authentication gates, checkpoints, atomic commits, and SUMMARY creation.

#### `get-shit-done/workflows/execute-plan.md`
The sub-workflow invoked by executor agents. Contains the per-plan execution logic including task execution, commit protocol, checkpoint protocols, deviation rules, and SUMMARY generation.

#### `get-shit-done/references/checkpoints.md`
Detailed reference for the three checkpoint types (`human-verify`, `decision`, `human-action`), automation patterns, and anti-patterns.

### Flow Trace

```
User: /gsd:execute-phase {phase}
         |
         v
execute-phase.md (orchestrator)
         |
         +-- init: load STATE.md, discover plans, group by waves
         |
         +-- handle_branching: git checkout -b (if phase/milestone strategy)
         |
         +-- discover_and_group_plans:
         |      gsd-tools phase-plan-index -> wave groupings
         |
         +-- For each wave:
         |      For each plan in wave:
         |           Spawn Task(subagent_type="gsd-executor",
         |                      isolation="worktree",
         |                      model=executor_model)
         |                |
         |                v
         |           gsd-executor.md
         |                |
         |                +-- load PLAN.md
         |                +-- execute_tasks (auto, checkpoint, tdd)
         |                +-- per-task atomic git commit
         |                +-- create SUMMARY.md
         |                +-- update STATE.md
         |                +-- return completion
         |
         +-- Post-wave: pre-commit hook validation (if parallel)
         |
         +-- checkpoint_handling: handle autonomous/non-autonomous plans
         |
         +-- verify_phase_goal: spawn gsd-verifier
         |
         +-- update_roadmap: mark phase complete
         |
         v
Phase complete / gaps found / next phase ready
```

### Key Code Snippets

**Wave grouping via gsd-tools (execute-phase.md step 3):**
```bash
PLAN_INDEX=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" phase-plan-index "${PHASE_NUMBER}")
# Returns: { phase, plans[], waves: {wave_num: [plan_ids]}, incomplete, has_checkpoints }
```

**Executor spawning with fresh context (execute-phase.md step 7):**
```bash
Task(
  subagent_type="gsd-executor",
  model="{executor_model}",
  isolation="worktree",
  prompt="
    <objective>Execute plan {plan_number} of phase {phase_number}-{phase_name}...</objective>
    <execution_context>
    @~/.claude/get-shit-done/workflows/execute-plan.md
    @~/.claude/get-shit-done/templates/summary.md
    @~/.claude/get-shit-done/references/checkpoints.md
    @~/.claude/get-shit-done/references/tdd.md
    </execution_context>
    <files_to_read>
    - {phase_dir}/{plan_file} (Plan)
    - .planning/PROJECT.md
    - .planning/STATE.md
    ...
    </files_to_read>
  "
)
```

**Executor deviation rules (gsd-executor.md):**
```markdown
**RULE 1: Auto-fix bugs** — Broken behavior, errors, race conditions
**RULE 2: Auto-add missing critical functionality** — Missing error handling, auth, validation
**RULE 3: Auto-fix blocking issues** — Missing deps, broken imports, wrong types
**RULE 4: Ask about architectural changes** — New DB table, schema change, library switch

Rule priority: RULE 4 STOP > Rules 1-3 auto-fix > unsure -> RULE 4
```

**Authentication gate pattern (gsd-executor.md):**
```markdown
# Auth errors during execution are NOT failures — they're expected interaction points.
# Protocol: Recognize -> STOP -> Create checkpoint:human-action -> Wait for user -> Retry
```

**Executor TDD flow (gsd-executor.md):**
```markdown
**RED:** Create test file -> write failing test -> run (MUST fail) -> commit: `test({phase}-{plan}): add failing test`
**GREEN:** Write minimal code to pass -> run (MUST pass) -> commit: `feat({phase}-{plan}): implement feature`
**REFACTOR:** Clean up -> tests pass -> commit: `refactor({phase}-{plan}): clean up`
```

### Notable Implementation Details

**1. Parallelization with worktree isolation**
Each executor runs with `isolation="worktree"`, which gives a fresh git worktree. This prevents parallel agents from contending over the same working directory. Commits use `--no-verify` flag to avoid pre-commit hook lock contention; the orchestrator runs hooks once after the entire wave completes.

**2. Three execution patterns (execute-plan.md)**
```bash
# Pattern A: Fully autonomous (no checkpoints) -> spawn single subagent
# Pattern B: Verify-only checkpoints -> segment-by-segment execution
# Pattern C: Decision checkpoints -> execute in main context
```

**3. Fresh context per subagent**
The orchestrator stays lean (~10-15% context) by only tracking paths and state. Each executor subagent gets a fresh 200k-token context, ensuring peak quality throughout execution. This is the "Nyquist Rule" in practice: re-sample context at natural boundaries (plan execution).

**4. Wave safety filter (execute-phase.md step 3)**
```bash
# Wave filter ensures lower waves complete before higher waves run:
# "If WAVE_FILTER is set and there are still incomplete plans in any lower wave...
#  STOP and tell the user to finish earlier waves first."
```

**5. Checkpoint types breakdown**
- `checkpoint:human-verify` (90%): After automation, user verifies visually/functionally
- `checkpoint:decision` (9%): User picks between architectural/technology options
- `checkpoint:human-action` (1%): Truly unavoidable manual steps (email 2FA, OAuth approval)

**6. Auto-mode checkpoint bypass (checkpoints.md)**
When `workflow._auto_chain_active` or `workflow.auto_advance` is true:
- `human-verify` auto-approves
- `decision` auto-selects first option
- `human-action` still stops (auth gates cannot be automated)

**7. Continuation is fresh-agent, not resume (execute-phase.md)**
> "Why fresh agent, not resume: Resume relies on internal serialization that breaks with parallel tool calls. Fresh agents with explicit state are more reliable."

**8. Cooldown checkpoint before dependent wave (execute-phase.md step 5b)**
Before spawning wave N+1, key-links from wave N artifacts are verified:
```bash
node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" verify key-links {plan}-PLAN.md
```
If prior-wave artifacts are NOT wired to current-wave expectations, a "Cross-Plan Wiring Gap" is flagged.

### Technical Debt and Shortcuts Observed

**1. No automatic wave deadlock detection**
If Plan A in Wave 1 depends on Plan B in Wave 1 (circular dependency), the system will hang waiting. The `depends_on` frontmatter field is trusted but not validated for cycles.

**2. Copilot runtime fallback is fragile (execute-phase.md step 1)**
The Copilot detection relies on a heuristic:
```bash
# Check for the @gsd-executor agent pattern or absence of Task() subagent API
# If Copilot detected: force sequential inline execution
```
This is fragile - runtime detection by feature presence is less reliable than an explicit flag.

**3. Missing SUMMARY.md is treated as failure, but executor may be mid-work**
Spot-checks verify SUMMARY exists and commits are present, but there's a window where the executor created files but hasn't yet written SUMMARY. The "classifyHandoffIfNeeded" Claude Code bug (false failure reporting) further complicates this.

**4. `verify_gap_plans` step is in verify-work.md, not execute-phase.md**
Gap-closure planning is triggered by `verify-work.md` after UAT diagnosis, not by `execute-phase.md`. This means the verification gap flow is separate from execution gap flow — if an executor encounters a gap during execution (not caught by verification), there's no automatic gap-closure routing.

**5. No automatic rollback on wave failure**
If an entire wave fails (all plans fail), there's no automatic rollback. User must manually assess and decide.

---

## Feature 5: Work Verification (UAT)

### Feature Name and Description

**Work Verification (UAT)** is the user acceptance testing workflow. It creates persistent UAT.md files that survive `/clear`, extracts testable deliverables from SUMMARY.md, walks the user through one test at a time, infers severity from natural language responses, and routes failures to gap diagnosis and fix planning. The key insight is that "User tests, Claude records" — this is not automated testing but conversational testing with persistent state.

### Core Implementation Files

#### `get-shit-done/workflows/verify-work.md`
The main UAT workflow. Orchestrates test presentation, response processing, severity inference, gap tracking, diagnosis, and fix planning.

#### `agents/gsd-verifier.md`
The verification agent for goal-backward phase verification (distinct from UAT but related). Verifies that phase goals are actually achieved in the codebase.

#### `get-shit-done/templates/UAT.md`
The persistent UAT session template with frontmatter, Current Test section, Tests section, Summary counts, and Gaps YAML.

#### `get-shit-done/workflows/verify-phase.md`
Called by `execute-phase.md` to verify phase goals after execution completes.

### Flow Trace

```
User: /gsd:verify-work {phase}
         |
         v
verify-work.md
         |
         +-- check_active_session: find existing UAT.md files
         |      If resumable session -> resume_from_file
         |      If new phase -> create_uat_file
         |
         +-- find_summaries: locate phase SUMMARY.md files
         |
         +-- extract_tests: parse deliverables -> test list
         |      +-- Cold-start smoke test injection
         |      (prepend if server/database/startup files modified)
         |
         +-- create_uat_file: write .planning/phases/{phase}/{num}-UAT.md
         |
         +-- Loop:
         |      present_test (via gsd-tools uat render-checkpoint)
         |      wait for user response
         |      process_response:
         |           "yes"/"y"/"ok"/"pass"/"next" -> pass
         |           "skip" -> skipped
         |           "blocked" -> blocked (infer blocked_by tag)
         |           anything else -> issue (infer severity)
         |      update UAT.md (batched writes)
         |      next test or complete_session
         |
         +-- complete_session:
         |      If issues -> diagnose_issues
         |      If no issues -> Phase complete
         |
         +-- diagnose_issues:
         |      Spawn parallel debug agents per issue
         |      Fill root_cause, artifacts, missing in Gaps YAML
         |
         +-- plan_gap_closure:
         |      Spawn gsd-planner in --gaps mode
         |      Spawn gsd-plan-checker to verify plans
         |      Revision loop (max 3 iterations)
         |
         v
Gap closure plans ready -> /gsd:execute-phase --gaps-only
```

### Key Code Snippets

**Test extraction from SUMMARY (verify-work.md step 3):**
```bash
# Extract from Accomplishments and User-facing changes in SUMMARY.md
# Focus on USER-OBSERVABLE outcomes, not implementation details
# Cold-start smoke test: prepend if any of these paths modified:
# server.ts, server.js, app.ts, database/*, db/*, seed/*, migrations/*, docker-compose*
```

**Response processing with severity inference (verify-work.md step 8):**
```markdown
# Infer from description:
# Contains: crash, error, exception, fails, broken, unusable -> blocker
# Contains: doesn't work, wrong, missing, can't -> major
# Contains: slow, weird, off, minor, small -> minor
# Contains: color, font, spacing, alignment, visual -> cosmetic
# Default if unclear: major
```

**UAT.md Gaps YAML structure (UAT.md template):**
```yaml
- truth: "[expected behavior from test]"
  status: failed
  reason: "User reported: [verbatim response]"
  severity: blocker | major | minor | cosmetic
  test: [N]
  root_cause: ""     # Filled by diagnosis
  artifacts: []      # Filled by diagnosis
  missing: []        # Filled by diagnosis
  debug_session: ""  # Filled by diagnosis
```

**Checkpoint rendering via gsd-tools (verify-work.md step 4):**
```bash
CHECKPOINT=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" uat render-checkpoint --file "$uat_path" --raw)
# Displays: Current Test section from UAT.md
```

**Diagnosis spawning (verify-work.md step 12):**
```bash
# Follow diagnose-issues workflow
# Spawn parallel debug agents for each issue
# Each agent: investigate root cause, return diagnosis
# Update UAT.md Gaps with root_cause, artifacts, missing
```

**Gap-closure planning loop (verify-work.md steps 13-16):**
```bash
# Spawn gsd-planner in gap_closure mode
# Spawn gsd-plan-checker to verify
# Revision loop: max 3 iterations
# If max iterations reached: offer 1) Force proceed 2) Provide guidance 3) Abandon
```

### Notable Implementation Details

**1. Persistent UAT file survives /clear**
The UAT.md file is written to `.planning/phases/{phase}/`, not memory. On `/clear`, the next invocation reads the UAT.md file and resumes from the first pending test. This is critical for long-running UAT sessions.

**2. Batched writes with safety net (verify-work.md update_rules):**
```markdown
| Section     | Rule           | When Written |
|-------------|----------------|--------------|
| Frontmatter | OVERWRITE      | Start, complete |
| Current Test | OVERWRITE    | On any file write |
| Tests.{N}   | OVERWRITE      | On any file write |
| Summary     | OVERWRITE      | On any file write |
| Gaps        | APPEND         | When issue found |
On context reset: File shows last checkpoint. Resume from there.
```

**3. Blocked tests do NOT become gaps**
```markdown
# "blocked" result means prerequisites aren't met (server not running, need physical device)
# These are NOT code issues — they're prerequisite gates
# They go into blocked_by tags but NOT into the Gaps section
```

**4. Severity inference from natural language — never asked**
The system NEVER asks "how severe is this?" It infers from the user's response. This keeps the conversational flow natural while capturing structured data.

**5. Two separate verification systems**
- `gsd-verifier.md` (agent): Goal-backward code verification after execution — checks actual codebase
- `verify-work.md` (workflow): Conversational UAT with the user — checks perceived behavior
These are complementary: verifier checks what was built, UAT checks if it works for the user.

**6. Parallel gap diagnosis**
When issues are found, debug agents are spawned in parallel (not sequentially) to minimize diagnosis time. Each agent investigates one gap independently.

**7. "Cold-start smoke test" injection**
Before testing, if any infrastructure files were modified (server, database, migrations, docker), a "Cold Start Smoke Test" is prepended to the test list. This catches bugs that only manifest on fresh start.

**8. UAT status lifecycle**
```
testing -> partial (if paused with pending/blocked tests) OR complete (all tests resolved)
         -> diagnosed (if issues found and diagnosed)
         -> resolved (if gap-closure completed the fixes)
```

### Technical Debt and Shortcuts Observed

**1. UAT tests are extracted from SUMMARY.md, which is Claude-generated**
SUMMARY.md is written by the executor agent, who may overstate accomplishments or miss edge cases. The verifier's job to catch this is separate (`gsd-verifier.md`). But UAT tests derived from SUMMARY could be missing important test cases the executor didn't think to mention.

**2. Diagnosis agents run independently — no cross-issue correlation**
Each gap gets its own debug agent running in parallel. But if two issues share a root cause (e.g., missing DB migration affects multiple features), each agent will discover the same root cause independently. No deduplication.

**3. Gap-closure planning is a separate workflow triggered after UAT**
The `verify-work.md` workflow handles diagnosis and gap-closure planning, but this is separate from the `execute-phase.md` execution flow. A gap found during UAT must be planned and executed through a different path than a gap found during execution.

**4. Max 3 revision iterations may not be enough for complex architectural issues**
If the checker finds fundamental issues with the gap-closure plan, the planner gets 3 attempts before the user is offered "Force proceed" or "Abandon." For architectural changes, this may not be sufficient.

**5. UAT.md does not track which plan introduced a test**
SUMMARY.md files come from specific plans, but the UAT doesn't record which plan each test came from. If a test fails, it's attributed to the phase but not the specific plan. This could make it harder to trace failures to specific commits.

---

## Feature 6: Quick Task Mode

### Feature Name and Description

**Quick Task Mode** is a fast-path for ad-hoc tasks that need GSD guarantees (atomic commits, state tracking) without the full planning ceremony. It supports three composable flags: `--discuss` (lightweight discussion before planning), `--research` (focused research before planning), and `--full` (plan-checking and post-execution verification). Quick tasks run in `.planning/quick/{id}-{slug}/` and update STATE.md's "Quick Tasks Completed" table.

### Core Implementation Files

#### `get-shit-done/workflows/quick.md`
The complete quick task workflow. Handles argument parsing, branching, discussion, research, planning, plan-checking, execution, verification, and STATE.md updates.

### Flow Trace

```
User: /gsd:quick {description} [--discuss] [--research] [--full]
         |
         v
quick.md
         |
         +-- Parse arguments: --discuss, --research, --full flags
         |      Composite flags create different banners
         |
         +-- init quick: gsd-tools init quick "${DESCRIPTION}"
         |      Returns: quick_id (YYMMDD-xxx), slug, quick_dir, task_dir
         |
         +-- handle_branching: git checkout -b if branch_name set
         |
         +-- mkdir .planning/quick/{quick_id}-{slug}/
         |
         +-- [If --discuss] discuss_phase:
         |      Identify 2-4 gray areas (concrete decision points)
         |      Present via AskUserQuestion (multi-select)
         |      Discuss each selected area (1-2 focused questions each)
         |      Write {quick_id}-CONTEXT.md (locked decisions)
         |
         +-- [If --research] research_phase:
         |      Spawn gsd-phase-researcher (single, not 4 parallel)
         |      Write {quick_id}-RESEARCH.md
         |
         +-- spawn_planner:
         |      Mode: quick (simple) or quick-full (strict)
         |      1-3 focused tasks, target ~30% (simple) or ~40% (full) context
         |      Write {quick_id}-PLAN.md
         |
         +-- [If --full] plan_checker_loop:
         |      Spawn gsd-plan-checker
         |      Revision loop: max 2 iterations
         |
         +-- spawn_executor:
         |      Execute all tasks, atomic commits
         |      Create {quick_id}-SUMMARY.md
         |      Note: Does NOT update ROADMAP.md (separate from planned phases)
         |
         +-- [If --full] verification:
         |      Spawn gsd-verifier
         |      Create {quick_id}-VERIFICATION.md
         |
         +-- update_STATE.md:
         |      Append to "Quick Tasks Completed" table
         |      Include Status column if --full
         |
         +-- final_commit: commit all artifacts
         |
         v
Quick task complete -> /gsd:quick {next}
```

### Key Code Snippets

**Flag composition (quick.md step 1):**
```bash
# Banners change based on active flags:
# --discuss --research --full: "GSD ► QUICK TASK (DISCUSS + RESEARCH + FULL)"
# --discuss --full: "GSD ► QUICK TASK (DISCUSS + FULL)"
# --discuss only: "GSD ► QUICK TASK (DISCUSS)"
# --research only: "GSD ► QUICK TASK (RESEARCH)"
# --full only: "GSD ► QUICK TASK (FULL MODE)"
```

**Discussion gray areas identification (quick.md step 4.5a):**
```markdown
# Domain-aware heuristic for gray areas:
# - Something users SEE -> layout, density, interactions, states
# - Something users CALL -> responses, errors, auth, versioning
# - Something users RUN -> output format, flags, modes, error handling
# - Something users READ -> structure, tone, depth, flow
# - Something being ORGANIZED -> criteria, grouping, naming, exceptions
# Each gray area: concrete decision point, not vague category
# Max 2 questions per area — this is lightweight, not deep dive
```

**Quick task CONTEXT.md structure (quick.md step 4.5d):**
```markdown
# Note: Omits <deferred> section (no codebase scouting, no phase scope to defer)
# Only includes:
# - <domain> (task boundary)
# - <decisions> (from discussion)
# - <specifics> (specific ideas/references)
# - <canonical_refs> (external specs if referenced)
```

**Quick planner constraints (quick.md step 5):**
```bash
# Mode: quick-full or quick (from --full flag)
# Constraints:
# - Create a SINGLE plan with 1-3 focused tasks
# - Quick tasks should be atomic and self-contained
# - Research findings available (if --research)
# - Target ~30% context (simple) or ~40% (full)
# - MUST generate must_haves (if --full)
# - Each task MUST have files, action, verify, done (if --full)
```

**Plan checker loop (quick.md step 5.5):**
```bash
# Only runs if --full is set
# Scope (quick-full mode): skips checks requiring ROADMAP phase goal
#   - Requirement coverage: Does the plan address task description?
#   - Task completeness: Do tasks have files/action/verify/done?
#   - Key links: Are referenced files real?
#   - Scope sanity: Appropriately sized for quick task (1-3 tasks)?
#   - must_haves derivation: Traceable to task description?
# Max 2 iterations (vs 3 for standard gap-closure)
```

**Quick executor (quick.md step 6):**
```bash
Task(
  subagent_type="gsd-executor",
  model="{executor_model}",
  isolation="worktree",
  prompt="Execute quick task ${quick_id}..."
)
# Constraints:
# - Execute all tasks in plan
# - Commit each task atomically
# - Create summary at: ${QUICK_DIR}/${quick_id}-SUMMARY.md
# - Do NOT update ROADMAP.md (quick tasks are separate)
```

**STATE.md update (quick.md step 7):**
```bash
# Quick Tasks Completed table format:
# --full mode: | # | Description | Date | Commit | Status | Directory |
# simple mode: | # | Description | Date | Commit | Directory |
# Append row: | ${quick_id} | ${DESCRIPTION} | ${date} | ${commit_hash} | ${VERIFICATION_STATUS} | [...]
```

### Notable Implementation Details

**1. ROADMAP.md is NOT updated for quick tasks**
Quick tasks are explicitly outside the phase/roadmap system. They don't advance phase state, don't appear in roadmap progress, and aren't tracked in the same milestone structure. This is by design — quick tasks are for "off-roadmap" work.

**2. Single focused researcher (vs 4 parallel in full phases)**
The quick workflow spawns ONE researcher, not four. This is appropriate because quick tasks are targeted (single feature or fix), not broad domain surveys.

**3. Discussion creates CONTEXT.md but is lightweight**
Unlike full `discuss-phase.md`, the quick discussion:
- Identifies only 2-4 gray areas (not exhaustive)
- Max 2 questions per area
- Skips `<deferred>` section (nothing to defer to in quick tasks)
- Max 2 iteration revision loop (vs 3 for full gap-closure)

**4. Fresh branch per quick task if `branch_name` is set**
```bash
git checkout -b "$branch_name" 2>/dev/null || git checkout "$branch_name"
# All quick-task commits stay on that branch
# User handles merge/rebase afterward
```

**5. Quick ID format: YYMMDD-xxx (2s Base36 precision)**
```bash
# Example: 260326-ab (2026-03-26, 2nd task of the day)
# Slug: lowercase, hyphens, max 40 chars
# Directory: .planning/quick/260326-ab-my-task-slug/
```

**6. Quick tasks can run mid-phase**
Validation only checks ROADMAP.md exists, not phase status. This allows quick tasks to address immediate needs without blocking planned phase work.

**7. `--full` mode provides quality near-planning without full ceremony**
```bash
# --full adds:
# - Plan checking (gsd-plan-checker, max 2 iterations)
# - Post-execution verification (gsd-verifier)
# - Must_haves in plan frontmatter
# - Status column in STATE.md
# But skips: multi-wave coordination, phase tracking, requirement IDs
```

### Technical Debt and Shortcuts Observed

**1. Quick tasks have no requirement traceability**
Since ROADMAP.md is not updated, there's no linkage between quick task work and project requirements. A quick task could implement something that contradicts roadmap requirements without any warning.

**2. No wave/dependency system for multiple quick tasks**
If a user runs multiple quick tasks in sequence, there's no dependency management. If task B depends on task A, the user must ensure ordering manually. Quick tasks are atomic and self-contained by design, but this limits their use for anything complex.

**3. Discussion phase is per-task, not accumulative**
If the user runs 5 quick tasks with `--discuss`, they answer the same gray-area questions for each. There's no shared context that remembers "for this project, we always use X approach" — each quick task starts fresh.

**4. No automatic cleanup of quick task directories**
Over time, `.planning/quick/` could accumulate many task directories. There's no archival or pruning mechanism. The STATE.md table shows the history, but the directories themselves remain.

**5. `isolation="worktree"` for executor but branch is on main repo**
The executor is spawned with `isolation="worktree"`, but if `branch_name` was set in the quick workflow, the branch was created on the main repo, not a worktree. The worktree isolation means the executor might not see the branch at all — it gets a fresh worktree on the current HEAD.

**6. Quick executor does not update codebase map**
The `execute-plan.md` step `update_codebase_map` is skipped for quick tasks. This means `.planning/codebase/*.md` can drift as quick tasks add new libraries or patterns without updating the codebase map.

---

## Cross-Cutting Observations

### 1. The Orchestrator Pattern is Consistent Across All Three Features

All three features follow the thin-orchestrator pattern: a workflow stays lean, spawns specialized subagents, and routes results. `execute-phase.md` orchestrates executors/verifiers, `verify-work.md` orchestrates planners/checkers/debuggers, `quick.md` orchestrates researchers/planners/executors/verifiers. The pattern is: "orchestrator tracks state, agents do work."

### 2. Fresh Context Windows at Every Boundary

Every subagent invocation (executor, verifier, planner, debugger) starts with a fresh context. The orchestrator passes file paths; the subagent reads the files with full context. This prevents context rot and maintains quality across long executions. This is the "Nyquist Rule" applied to agentic workflows.

### 3. Three-Layer State Machine Pattern

The GSD system uses a consistent three-layer state pattern:
- **Layer 1**: Artifact files (PLAN.md, SUMMARY.md, UAT.md, VERIFICATION.md) — the persistent record
- **Layer 2**: STATE.md / ROADMAP.md — aggregated project state
- **Layer 3**: Git commits — the immutable, bisect-friendly audit trail

Every feature respects all three layers.

### 4. Checkpoint Philosophy

The three checkpoint types (`human-verify`, `decision`, `human-action`) map to fundamentally different interaction patterns:
- **human-verify**: Claude automated, human confirms (90%)
- **decision**: Human chooses direction (9%)
- **human-action**: Claude hit an auth gate, needs human credentials (1%)

The key insight is that pre-planned checkpoints are rare (`human-action` only) — most checkpoints are created dynamically after Claude attempts automation.

### 5. Error Classification is Structured

Errors are classified as:
- **Bugs** (Rule 1): Broken behavior — auto-fixed
- **Missing Critical** (Rule 2): Missing essential functionality — auto-fixed
- **Blocking** (Rule 3): Missing prerequisites — auto-fixed
- **Architectural** (Rule 4): Structural changes — user decision required

This classification drives whether execution pauses for human input or continues autonomously.

### 6. Gap Closure is a First-Class Feature

Gaps found during UAT (or verification) are not just reported — they flow through a structured gap-closure pipeline:
1. Diagnose (parallel debug agents)
2. Plan (gsd-planner in --gaps mode)
3. Check (gsd-plan-checker)
4. Revise (max 3 iterations)
5. Execute (execute-phase --gaps-only)
6. Re-verify

This is a complete feedback loop, not just a bug report.
