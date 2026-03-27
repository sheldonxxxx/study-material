# GSD Features Deep Dive

**Synthesized from:** 6 feature batch research files
**Date:** 2026-03-26
**Organized by:** Priority (core first, then secondary)

---

## Document Purpose

This document synthesizes all feature research into a coherent narrative for understanding the GSD (Get Shit Done) system. It is organized by feature priority, with each feature containing: description, key implementation insights, notable patterns, and technical debt.

---

# Part I: Core Features

These 7 features form the primary GSD workflow. A project follows this lifecycle:

```
new-project → discuss-phase → plan-phase → execute-phase → verify-work
                         ↑                                           ↓
                         ←←←←←← (if gaps found) ←←←←←←←←←←←←←←←←←←←←
```

---

## Feature 1: Project Initialization

**Priority:** Core
**Entry point:** `/gsd:new-project`

### Description

Single-command project setup that transforms ideas into structured plans via guided questions, research, and roadmap generation. This is the highest-leverage moment in any project — deep questioning here means better plans, better execution, better outcomes.

### Key Implementation Insights

**Flow:** Setup → Brownfield detection → Deep questioning (or auto-mode PRD extraction) → Write PROJECT.md → Workflow preferences → Optional research (4 parallel agents) → Requirements definition → Roadmap creation → Commit

**Parallel Research Pattern:** When research is enabled, 4 `gsd-project-researcher` agents run in parallel (Stack, Features, Architecture, Pitfalls), then `gsd-research-synthesizer` aggregates into SUMMARY.md.

**Brownfield Detection:** If existing code without a `.planning/codebase/` map exists, the workflow offers to run `/gsd:map-codebase` first. This prevents retrofitting GSD onto existing projects without understanding them.

**Sub-Repo Detection:** Uses `find . -maxdepth 1 -type d -not -name ".*" -not -name "node_modules" -exec test -d "{}/.git" \; -print` to detect multi-repo workspaces and configure `planning.sub_repos`.

**Auto-mode PRD Express Path:** When `--auto @prd.md` is provided, extracts requirements directly from the document without interactive questioning. Critical for CI/CD pipelines and experienced users.

### Notable Patterns

- Config is created BEFORE knowing greenfield vs brownfield — defaults are applied automatically
- Model profile set in config may not match what user would have chosen interactively
- No rollback if parallel researchers partially fail — synthesizer runs with incomplete results

### Technical Debt

- **Partial research failure:** If one of 4 parallel researchers fails, synthesizer still runs with incomplete data
- **Config initialization order:** Config created before greenfield/brownfield determination

---

## Feature 2: Phase Discussion

**Priority:** Core
**Entry point:** `/gsd:discuss-phase {phase}`

### Description

Captures implementation decisions and preferences before planning, producing CONTEXT.md that locks user decisions and feeds researcher/planner. A thinking partnership that captures decisions, not implementation.

### Key Implementation Insights

**Gray Areas:** The workflow identifies ambiguities that would change implementation. Each gray area is a concrete decision point, not a vague category. Max 2 questions per area for lightweight discussion.

**Prior Decisions Carry-Forward:** Builds `<prior_decisions>` from earlier phases so decisions made in Phase 2 don't get re-asked in Phase 5. Annotates gray areas with "You chose X in Phase N".

**Advisor Mode:** If `USER-PROFILE.md` exists with `vendor_philosophy`, calibrates research depth:
- `conservative/thorough-evaluator` → full_maturity (3-5 options)
- `opinionated` → minimal_decisive (1-2 options)
- `pragmatic-fast` → standard (2-4 options)

**Text Mode Support:** When `workflow.text_mode: true` or `--text` flag provided, replaces ALL `AskUserQuestion` calls with numbered lists. Required for Claude Code remote sessions (`/rc` mode).

**DISCUSSION-LOG.md:** Full Q&A preserved separately from CONTEXT.md for compliance audits. Not consumed by downstream agents.

### Notable Patterns

- Scope creep guard: If user suggests scope creep, redirect to backlog with "Want me to note it for the roadmap backlog?"
- Canonical refs accumulation: Actively collects references throughout discussion (from ROADMAP, REQUIREMENTS, user mentions, scout findings)
- Batch mode: `--batch, --batch=N, --batch N` allows grouping 2-5 questions per turn

### Technical Debt

- **Deep nesting warning:** `AskUserQuestion` does not work correctly in nested subcontexts (Claude Code issue #1009). Workflow must EXIT and let user re-invoke — cannot chain nested calls.
- **Auto mode:** Logs choices without real discussion — just captures what would have been chosen

---

## Feature 3: Phase Planning

**Priority:** Core
**Entry point:** `/gsd:plan-phase {phase}`

### Description

Multi-agent planning with research synthesis, XML-structured plan creation, and automated verification against requirements. Creates 2-3 atomic task plans per phase. Orchestrates gsd-phase-researcher, gsd-planner, and gsd-plan-checker with revision loop (max 3 iterations).

### Key Implementation Insights

**10-Dimensional Plan Checking:**
1. Requirement Coverage
2. Task Completeness
3. Dependency Correctness
4. Key Links Planned
5. Scope Sanity
6. Verification Derivation
7. Context Compliance (locked decisions honored)
8. Nyquist Compliance (automated testing requirements)
9. Cross-Plan Data Contracts
10. CLAUDE.md Compliance

**PRD Express Path:** When `--prd path/to/prd.md` provided, skips discuss-phase entirely and generates CONTEXT.md from PRD. All PRD requirements become locked decisions.

**Requirements Coverage Gate:** After plans pass checker, verifies ALL phase REQ-IDs appear in at least one plan's requirements field. 100% coverage required.

**Wave Assignment:** Plans include `wave` frontmatter field that enables parallel execution. Wave 1 plans have no dependencies; Wave 2 depends on Wave 1; etc.

**Gap Closure Mode:** Triggered by `--gaps` flag. Reads gaps from VERIFICATION.md or UAT.md, groups by artifact/concern/dependency, creates fix tasks, assigns waves.

### Notable Patterns

- **Mode injection:** One planner agent handles standard, gap_closure, reviews, and revision modes via mode-specific prompt injection
- **Wave 0 for tests:** If task has `tdd="true"` and no test exists, specifies `MISSING — Wave 0 must create {test_file} first`
- **CLAUDE.md compliance:** Plans must respect project-specific conventions; checker verifies forbidden libraries aren't used

### Technical Debt

- **Windows MCP stdio deadlock:** Common on Windows with MCP servers. Workaround: force-kill, clean orphaned processes, reduce MCP servers, retry with `--skip-research`
- **--auto chain uses --no-transition hack:** Uses Skill tool (not Task/Task) to avoid nested agent runtime freezes

---

## Feature 4: Wave-Based Phase Execution

**Priority:** Core
**Entry point:** `/gsd:execute-phase {phase}`

### Description

Parallel plan execution with fresh 200k-token contexts per plan, atomic git commits per task, and automatic dependency management across waves. The orchestrator stays lean (~10-15% context) and spawns fresh subagents for each plan.

### Key Implementation Insights

**Fresh Context Per Subagent:** Each executor subagent gets a fresh 200k-token context. This is the "Nyquist Rule" in practice: re-sample context at natural boundaries (plan execution).

**Worktree Isolation:** Each executor runs with `isolation="worktree"`, preventing parallel agents from contending over the same working directory.

**Parallel Agent --no-verify:** Pre-commit hooks cannot run concurrently across parallel agents. The deadlock is cleared by deferring hook validation to after all agents complete.

**Three Execution Patterns:**
- Pattern A: Fully autonomous (no checkpoints) → spawn single subagent
- Pattern B: Verify-only checkpoints → segment-by-segment execution
- Pattern C: Decision checkpoints → execute in main context

**Wave Safety Filter:** If lower waves are incomplete, higher waves do not start. User is told to finish earlier waves first.

**Cooldown Checkpoint:** Before spawning wave N+1, key-links from wave N artifacts are verified via `gsd-tools verify key-links`. If prior-wave artifacts are NOT wired to current-wave expectations, a "Cross-Plan Wiring Gap" is flagged.

**Executor Deviation Rules:**
- RULE 1: Auto-fix bugs (broken behavior, errors, race conditions)
- RULE 2: Auto-add missing critical functionality (missing error handling, auth, validation)
- RULE 3: Auto-fix blocking issues (missing deps, broken imports, wrong types)
- RULE 4: Ask about architectural changes (new DB table, schema change, library switch)

### Notable Patterns

- **Auth gates:** Authentication errors during execution are NOT failures — they create `checkpoint:human-action`. Protocol: Recognize → STOP → Create checkpoint → Wait for user → Retry
- **Continuation is fresh-agent, not resume:** Resume relies on internal serialization that breaks with parallel tool calls. Fresh agents with explicit state are more reliable.
- **Auto-mode checkpoint bypass:** When `workflow._auto_chain_active` or `workflow.auto_advance` is true, `human-verify` auto-approves, `decision` auto-selects first option

### Technical Debt

- **No automatic wave deadlock detection:** If Plan A in Wave 1 depends on Plan B in Wave 1 (circular dependency), system hangs. `depends_on` frontmatter is trusted but not validated for cycles.
- **Copilot runtime fallback is fragile:** Detection relies on heuristic (`@gsd-executor` pattern absence). Less reliable than explicit flag.
- **No automatic rollback on wave failure:** User must manually assess if entire wave fails.

---

## Feature 5: Work Verification (UAT)

**Priority:** Core
**Entry point:** `/gsd:verify-work {phase}`

### Description

User acceptance testing workflow. Creates persistent UAT.md files that survive `/clear`, extracts testable deliverables from SUMMARY.md, walks the user through one test at a time, infers severity from natural language responses, and routes failures to gap diagnosis and fix planning.

### Key Implementation Insights

**Persistent UAT File:** Written to `.planning/phases/{phase}/`, not memory. On `/clear`, next invocation reads UAT.md and resumes from first pending test.

**Severity Inference from Natural Language:** System NEVER asks "how severe is this?" Infers from response:
- Contains: crash, error, exception, fails, broken, unusable → blocker
- Contains: doesn't work, wrong, missing, can't → major
- Contains: slow, weird, off, minor, small → minor
- Contains: color, font, spacing, alignment, visual → cosmetic
- Default if unclear → major

**Two Separate Verification Systems:**
- `gsd-verifier.md` (agent): Goal-backward code verification after execution — checks actual codebase
- `verify-work.md` (workflow): Conversational UAT with user — checks perceived behavior

**Cold-Start Smoke Test:** Before testing, if infrastructure files modified (server, database, migrations, docker), a smoke test is prepended to catch fresh-start bugs.

**Parallel Gap Diagnosis:** When issues found, debug agents spawned in parallel (not sequentially). Each investigates one gap independently.

**Gap-Closure Pipeline:**
1. Diagnose (parallel debug agents)
2. Plan (gsd-planner in --gaps mode)
3. Check (gsd-plan-checker)
4. Revise (max 3 iterations)
5. Execute (execute-phase --gaps-only)
6. Re-verify

### Notable Patterns

- **Batched writes with safety net:** Sections have overwrite/append rules. UAT.md survives context resets.
- **Blocked tests do NOT become gaps:** Blocked means prerequisites unmet (server not running, need physical device). Not code issues. Goes into `blocked_by` tags, not Gaps section.
- **UAT status lifecycle:** testing → partial (if paused) OR complete → diagnosed (if issues found) → resolved (if gap-closure completed)

### Technical Debt

- **UAT tests from SUMMARY.md (Claude-generated):** Executor may overstate accomplishments or miss edge cases. Tests derived from SUMMARY could be incomplete.
- **No cross-issue correlation:** If two issues share root cause, each debug agent discovers it independently. No deduplication.
- **Gap-closure is separate workflow:** Gap found during UAT must be planned/executed differently than gap found during execution.

---

## Feature 6: Quick Task Mode

**Priority:** Core
**Entry point:** `/gsd:quick {description} [--discuss] [--research] [--full]`

### Description

Fast-path for ad-hoc tasks needing GSD guarantees (atomic commits, state tracking) without full planning ceremony. Supports three composable flags: `--discuss` (lightweight discussion), `--research` (focused research), `--full` (plan-checking and post-execution verification).

### Key Implementation Insights

**Quick tasks run in `.planning/quick/{id}-{slug}/`**, not in phase structure. Updates STATE.md's "Quick Tasks Completed" table.

**ROADMAP.md is NOT updated:** Quick tasks are explicitly outside the phase/roadmap system. They don't advance phase state, don't appear in roadmap progress.

**Single focused researcher:** Spawns ONE researcher (not four like full phases) because quick tasks are targeted, not broad surveys.

**Quick ID format:** YYMMDD-xxx (2s Base36 precision). Example: 260326-ab (2026-03-26, 2nd task of the day).

**--full mode adds:** Plan checking (gsd-plan-checker, max 2 iterations), post-execution verification (gsd-verifier), must_haves in plan frontmatter, status column in STATE.md.

**Discussion is lightweight:** Only 2-4 gray areas, max 2 questions per area, skips `<deferred>` section, max 2 revision iterations.

### Notable Patterns

- Quick tasks can run mid-phase — validation only checks ROADMAP.md exists, not phase status
- Fresh branch per quick task if `branch_name` is set
- 1-3 focused tasks, target ~30% context (simple) or ~40% (full)

### Technical Debt

- **No requirement traceability:** No linkage between quick task work and project requirements. Could contradict roadmap requirements.
- **No wave/dependency system:** If task B depends on task A, user must order manually.
- **Discussion is per-task:** No shared context remembering "for this project, we always use X approach"
- **No automatic cleanup:** `.planning/quick/` accumulates directories over time

---

## Feature 7: Codebase Mapping

**Priority:** Core
**Entry point:** `/gsd:map-codebase`

### Description

Analyzes existing codebase before new-project to understand stack, architecture, conventions, and concerns. Spawns parallel agents for each analysis area.

### Key Implementation Insights

**Parallel Agent Spawning:** 4 `gsd-codebase-mapper` agents run in parallel:
- Agent 1 (Tech Focus) → STACK.md, INTEGRATIONS.md
- Agent 2 (Architecture Focus) → ARCHITECTURE.md, STRUCTURE.md
- Agent 3 (Quality Focus) → CONVENTIONS.md, TESTING.md
- Agent 4 (Concerns Focus) → CONCERNS.md

**Runtime Capability Detection:** Before spawning agents, detects whether Task tool is available. If unavailable (e.g., Antigravity, Gemini CLI, Codex), performs mapping sequentially in-context.

**Secret Scanning:** Before committing generated documents, runs secret pattern scan:
```bash
grep -E '(sk-[a-zA-Z0-9]{20,}|sk_live_|ghp_|gho_|glpat-|AKIA|xox[baprs]-|
-----BEGIN.*PRIVATE KEY|eyJ[a-zA-Z0-9_-]+\.eyJ[a-zA-Z0-9_-]+\.)' \
  .planning/codebase/*.md
```

**Forbidden Files:** Explicitly never read or quote from: `.env`, `credentials.*`, `secrets.*`, `*.pem`, `*.key`, SSH private keys, NPMRC tokens.

### Notable Patterns

- **Document quality over brevity:** Agent instructions: "A 200-line TESTING.md with real patterns is more valuable than a 74-line summary"
- **Sequential fallback:** Gracefully degrades to sequential in-context mapping when Task tool unavailable

### Technical Debt

None observed — feature appears well-designed with proper separation of concerns.

---

# Part II: Secondary Features

These 10 features support the core workflow with specialized capabilities.

---

## Feature 8: Multi-Agent Orchestration

**Priority:** Secondary

### Description

Thin orchestrator pattern that spawns specialized agents and routes results between stages. Keeps main context at 30-40%. The system defines 17 specialized agents.

### Key Implementation Insights

**Agent Registry:**
- gsd-planner, gsd-executor, gsd-verifier, gsd-debugger
- gsd-project-researcher, gsd-phase-researcher, gsd-research-synthesizer
- gsd-codebase-mapper, gsd-roadmapper
- gsd-plan-checker, gsd-integration-checker, gsd-nyquist-auditor
- gsd-assumptions-analyzer, gsd-advisor-researcher
- gsd-ui-auditor, gsd-ui-researcher, gsd-ui-checker
- gsd-user-profiler

**Model Tiering by Task Complexity:**
| Agent | quality | balanced | budget | inherit |
|-------|---------|----------|--------|---------|
| gsd-planner | opus | opus | sonnet | inherit |
| gsd-executor | opus | sonnet | sonnet | inherit |
| gsd-phase-researcher | opus | sonnet | haiku | inherit |
| gsd-codebase-mapper | sonnet | haiku | haiku | inherit |
| gsd-verifier | sonnet | sonnet | haiku | inherit |

**Agent Skills System:** `buildAgentSkillsBlock()` injects project-specific skills into any agent type via config.agent_skills.

### Notable Patterns

- **Thin orchestrator:** Coordinates by spawning agents, routing results, validating outputs — never implements features directly
- **Non-blocking:** Agents spawn with `run_in_background=true` and use `TaskOutput` for true parallelism
- **Graceful degradation:** When agent spawn fails or produces incomplete output, workflow notes failures and continues

### Technical Debt

- **Context contamination:** GSD returns `"inherit"` for opus-tier agents, causing them to use whatever opus version user has configured. Workaround creates non-deterministic behavior.

---

## Feature 9: Context Engineering System

**Priority:** Secondary

### Description

File-based context management using PROJECT.md, research/, REQUIREMENTS.md, ROADMAP.md, STATE.md, PLAN.md, SUMMARY.md to prevent context rot across sessions.

### Key Implementation Insights

**Context File Structure:**
```
.planning/
├── PROJECT.md          # Project definition, value, constraints
├── ROADMAP.md         # Phase definitions with goals
├── STATE.md           # Current position, metrics, accumulated context
├── REQUIREMENTS.md    # Requirement traceability
├── config.json        # Planning behavior configuration
├── MILESTONES.md      # Milestone tracking
├── codebase/          # Codebase analysis maps
├── phases/
│   └── XX-name/
│       ├── XX-CONTEXT.md       # User decisions from discuss-phase
│       ├── XX-RESEARCH.md      # Technical research
│       ├── XX-PLAN.md          # Implementation plan
│       ├── XX-SUMMARY.md       # Execution results
│       └── XX-UAT.md           # User acceptance testing
├── todos/
└── archive/
```

**Summary Dependency Graph:** PLAN.md and SUMMARY.md use YAML frontmatter with `requires/provides/affects` fields enabling transitive context closure during planning.

**Context Budget Enforcement:** ~50% context target per plan. Each plan has 2-3 tasks maximum and should complete within ~50% of token budget.

**Size Constraint:** STATE.md explicitly limited to 100 lines. It's a digest, not an archive.

### Notable Patterns

- **File-based state prevents loss:** All context persists across sessions in `.planning/`
- **Separation of concerns:** Each file type has a distinct role
- **Auto-detection of gitignored .planning/:** If `.planning/` is gitignored, `commit_docs` automatically set to `false`

### Technical Debt

- **Potential staleness:** Context files can become stale if user manually edits code without updating SUMMARY, multiple sessions run concurrently, or plan-phase doesn't fully consume prior summaries

---

## Feature 10: UI Design Workflow

**Priority:** Secondary

### Description

UI phase contract generation (UI-SPEC.md) and retroactive 6-pillar visual audit of implemented frontend code.

### Key Implementation Insights

**Two Workflows:**
- `ui-phase.md` (prospective): Creates UI-SPEC.md contract before implementation
- `ui-review.md` (retrospective): Audits implemented code

**6-Pillar Audit:**
1. Copywriting — CTA must be verb+noun
2. Visuals — Focal point declared, icon-only needs aria-label
3. Color — 60/30/10 split, accent reserved-for
4. Typography — Max 4 font sizes, max 2 weights, line height declared
5. Spacing — Must be multiple of 4
6. Registry Safety — Third-party blocks vetted via `npx shadcn view`

**Shadcn Initialization Gate:** For React/Next.js/Vite, if components.json not found, asks user to initialize shadcn. User pastes preset string from ui.shadcn.com/create.

**Registry Vetting Gate:** For each third-party block declared, runs `npx shadcn view {block} --registry {registry_url}` and scans for suspicious patterns (network access, `process.env`, `eval`, dynamic imports from external URLs).

**Max 2 Revision Iterations:** With force-approve escape hatch.

### Notable Patterns

- **Upstream pre-population:** Researcher only asks questions not already answered by CONTEXT.md, RESEARCH.md, REQUIREMENTS.md
- **Tool strategy priority:** Codebase grep → Context7 → Exa MCP → Firecrawl MCP → WebSearch
- **Gitignore gate:** Before any screenshot, creates `.gitignore` preventing binary files from reaching git

### Technical Debt

- **No UI-SPEC versioning:** If design changes mid-execution, no mechanism to track evolution
- **Screenshot diff not stored:** No visual diff between audits
- **No CSS variable auditing:** Grep-based audits don't verify custom CSS properties match declared tokens
- **ui-review duplicates auditor logic:** Both contain same 6-pillar definitions

---

## Feature 11: Git Integration & Atomic Commits

**Priority:** Secondary

### Description

Per-task atomic commits with surgical traceability, branch strategies (none/phase/milestone), and PR creation from verified work.

### Key Implementation Insights

**Core Principle:** "Commit outcomes, not process." Git log should read like a changelog of what shipped, not a diary of planning activity.

**Commit Point Decision:**
| Event | Commit? | Why |
|-------|---------|-----|
| BRIEF + ROADMAP created | YES | Project initialization |
| PLAN.md created | NO | Intermediate |
| RESEARCH.md created | NO | Intermediate |
| **Task completed** | **YES** | Atomic unit of work |
| **Plan completed** | **YES** | Metadata commit |
| Handoff created | YES | WIP state preserved |

**Branch Strategies:**
- `none`: All commits on current branch
- `phase`: Branch per phase (`gsd/phase-{phase}-{slug}`)
- `milestone`: Branch per milestone (`gsd/{milestone}-{slug}`)

**PR Branch Filtering:** `/gsd:pr-branch` creates clean PR branch by classifying commits:
- Code commits: touches at least one non-.planning/ file
- Planning-only: only .planning/ files
- Mixed: cherry-pick with `.planning/` removed

**commit-docs CLI:** `gsd-tools.cjs commit` handles all logic for whether to commit (checks `commit_docs` config, checks `.planning/` gitignore status).

### Notable Patterns

- **Parallel agent --no-verify shortcut:** Acknowledges pre-commit hooks can't run concurrently. Defers validation to after all agents complete.
- **Cherry-pick with path filtering:** Preserves original commit messages when creating clean PR branches
- **sub_repos auto-sync:** Config may be rewritten when repos change on disk

### Technical Debt

- **Cherry-pick is O(n):** No batch optimization for large histories
- **No merge commit handling:** Algorithm only handles linear commits
- **No force-push handling:** If branch exists on remote with different content, `git push` fails
- **No git hooks for GSD-managed repos:** Hooks directory contains hooks for GSD tool itself, not for projects using GSD

---

## Feature 12: Security Hardening

**Priority:** Secondary

### Description

Defense-in-depth security including path traversal prevention, prompt injection detection, safe JSON parsing, and shell argument validation.

### Key Implementation Insights

**gsd-prompt-guard.js:** PreToolUse hook scanning content being written to `.planning/` files. Advisory-only — appends warning to context but does NOT block.

**Injection Patterns (17):**
- `ignore previous instructions`, `disregard previous`, `forget instructions`
- `you are now a/an/the`, `pretend you are`
- `from now on, you are`
- `print/output/reveal your prompt`
- `<system>`, `<assistant>`, `<human>` tags
- `[SYSTEM]`, `[INST]`, `<<SYS>>`

**Path Traversal Prevention:** Symlink resolution + containment check. Handles macOS `/var -> /private/var` chains.

**Sanitization:** Neutralizes control characters:
- Zero-width Unicode stripped
- XML/HTML tags mimicking system boundaries neutralized (`<system>` → `＜system-text＞`)
- `[SYSTEM]` → `[$1-TEXT]`

**Shell Argument Validation:** Rejects null bytes and command substitution (`$(`, backtick). Allows `$VAR` but blocks `$(cmd)`.

**Phase Number Validation:** Regex-only (no eval). Prevents ReDoS.

### Notable Patterns

- **Silent failure in hook:** If JSON parsing fails or stdin times out, `process.exit(0)` passes through. Hook NEVER blocks legitimate operations.
- **Hook is self-contained:** Has own inline copy of patterns rather than importing from `security.cjs`
- **`act as a plan` explicitly allowed:** Negative lookahead prevents false positives on legitimate GSD phrase

### Technical Debt

- **Hook advisory-only:** Cannot block attacks — only surfaces warnings
- **Pattern duplication:** Patterns exist in two places (security.cjs and gsd-prompt-guard.js) with no shared source
- **No security.cjs test file:** 380-line security library has no dedicated test
- **`sanitizeForPrompt` incomplete:** Doesn't handle homoglyph attacks, URL-encoded tags, other encoding tricks

---

## Feature 13: Session Management

**Priority:** Secondary

### Description

Pause/resume work with HANDOFF.json persistence, session reports, and health checks for `.planning/` directory integrity.

### Key Implementation Insights

**Dual-Format Handoff:**
- Machine-readable: `.planning/HANDOFF.json`
- Human-readable: `.planning/phases/XX-name/.continue-here.md`

**HANDOFF.json Structure:**
```json
{
  "version": "1.0",
  "timestamp": "{timestamp}",
  "phase": "{phase_number}",
  "plan": {current_plan_number},
  "task": {current_task_number},
  "status": "paused",
  "completed_tasks": [...],
  "remaining_tasks": [...],
  "blockers": [...],
  "next_action": "{specific first action}",
  "context_notes": "{mental state}"
}
```

**Health Check Error Taxonomy:**
| Code | Severity | Description | Repairable |
|------|----------|-------------|------------|
| E001-E003 | error | Directory/file not found | No |
| E004-E005 | error | STATE.md missing/config parse error | Yes |
| W001-W009 | warning | Various structural issues | Varies |
| I001 | info | Plan without SUMMARY | No |

**STATE.md content is NOT repairable:** Too risky to overwrite session history. Only structural absence is repairable.

### Notable Patterns

- **Placeholder content detection:** pause-work checks for false completions in SUMMARY files
- **Windows-specific:** Detects stale Claude Code task directories (older than 24h)

### Technical Debt

- **Resume-work not implemented as workflow:** Appears delegated entirely to command-level implementation
- **No automatic cleanup of old session reports:** Accumulates without rotation policy

---

## Feature 14: Debugging & Forensics

**Priority:** Secondary

### Description

Systematic debugging with persistent state (`/gsd:debug`) and post-mortem investigation of failed workflow runs (`/gsd:forensics`).

### Key Implementation Insights

**gsd-debugger.md (1,374 lines):** Most methodologically rich agent in codebase.

**File-based session persistence:** `.planning/debug/{slug}.md` with strict immutability rules:
- Symptoms section: IMMUTABLE after gathering
- Eliminated section: APPEND only (never overwrite eliminated hypotheses)
- Current Focus: OVERWRITE on each update

**Status State Machine:**
```
gathering → investigating → fixing → verifying → awaiting_human_verify → resolved
                  ↑            |           |                 |
                  |____________|___________|_________________|
```

**Knowledge Base:** `.planning/debug/knowledge-base.md` — append-only record of resolved sessions. New sessions check for keyword overlap at investigation start.

**Hypothesis Testing Discipline:**
- Falsifiability required: specific, testable claims
- One hypothesis at a time
- Evidence quality rated: strong vs weak
- Binary search / divide-and-conquer for large codebases

**forensics.md:** Explicitly read-only — only writes forensic report.

**Anomaly Detection Patterns:**
- Stuck Loop: Same file in 3+ consecutive commits within short time window
- Missing Artifact: Phase appears complete but lacks expected artifacts
- Abandoned Work: Large gap between last commit and current time
- Scope Drift: Recent commits touch files outside phase's expected scope

### Notable Patterns

- **"User = Reporter, Claude = Investigator":** Explicitly instructed to treat own code as foreign when debugging code it wrote
- **Red flag phrases:** "seems to work", "I think it's fixed" flagged as trust-building failures
- **`goal: find_root_cause_only` mode:** Allows forensics to use debugger without full fix cycle

### Technical Debt

- **Knowledge base matching is keyword overlap:** Could surface false positives
- **No integration with actual debugging tools:** Purely code reading + logging
- **Orphaned worktree detection fragile:** Depends on user not having legitimate worktrees

---

## Feature 15: Milestone & Phase Management

**Priority:** Secondary

### Description

Complete milestone lifecycle (new, audit, complete) and phase management (add, insert, remove, validate) with gap closure planning.

### Key Implementation Insights

**Milestone Completion (complete-milestone.md, 700+ lines):**
1. verify_readiness — comprehensive phase status, requirements completion
2. gather_stats — git log, diff stat, LOC counting
3. extract_accomplishments — one-liner extraction from SUMMARY files
4. evolve_project_full_review — PROJECT.md evolution at milestone boundary
5. archive_milestone — delegates to `gsd-tools.cjs milestone complete`
6. reorganize_roadmap_and_delete_originals — groups completed phases
7. handle_branches — squash merge vs merge with history
8. git_tag — `git tag -a v[X.Y]`
9. write_retrospective — append to RETROSPECTIVE.md

**3-Source Cross-Reference for Requirements:**
| VERIFICATION.md | SUMMARY Frontmatter | REQUIREMENTS.md | Final Status |
|----------------|---------------------|-----------------|--------------|
| passed | listed | [x] | satisfied |
| passed | listed | [ ] | satisfied (update checkbox) |
| passed | missing | any | partial |
| gaps_found | any | any | unsatisfied |
| gaps_found | any | any | unsatisfied |
| missing | listed | any | partial |
| missing | missing | any | unsatisfied |

**Gap-to-Phase Clustering Rules:**
- Same affected phase → combine
- Same subsystem → combine
- Dependency order (fix stubs before wiring)
- 2-4 tasks per phase

### Notable Patterns

- **Delegation pattern:** CLI handles file I/O, workflow provides judgment calls
- **`getMilestonePhaseFilter` shared:** Between `cmdPhasesList` and `cmdMilestoneComplete` ensures consistent milestone scoping
- **Task counting fallback chain:** `**Tasks:** N` → XML `<task` tags → `## Task N` markdown headers

### Technical Debt

- **complete-milestone.md extremely long:** 700+ lines with deep nesting
- **Milestone archival deletes originals:** If archival fails mid-way, could lose state
- **Health checks and milestone audits do validation differently:** No shared validation infrastructure

---

## Feature 16: Multi-Project Workspaces

**Priority:** Secondary

### Description

Isolated workspaces with repo copies (worktrees or clones) for parallel milestone work, plus workstream management for namespaced parallel tracks.

### Key Implementation Insights

**Two Sub-Features:**
1. **Physical Workspaces** (`/gsd:new-workspace`): Creates isolated directory structures with actual git copies
2. **Workstreams** (`/gsd:workstreams`): Virtual milestone namespaces within a single repo

**Workstream Directory Structure:**
```
.planning/
├── PROJECT.md          # Shared
├── config.json         # Shared
├── milestones/         # Shared
├── codebase/           # Shared
├── active-workstream   # Points to current ws
└── workstreams/
    ├── feature-a/
    │   ├── STATE.md
    │   ├── ROADMAP.md
    │   ├── REQUIREMENTS.md
    │   └── phases/
    └── feature-b/
        ...
```

**--ws Flag Resolution Priority:**
1. `--ws <name>` flag (explicit, highest)
2. `GSD_WORKSTREAM` env var
3. `.planning/active-workstream` file
4. `null` — flat mode

**Migration Logic:** `migrateToWorkstreams` moves ROADMAP.md, STATE.md, REQUIREMENTS.md, phases/ from flat to workstream structure. Shared files stay in place.

### Notable Patterns

- **Migration safety:** Rollback on failure
- **Slug sanitization:** Names sanitized to lowercase hyphenated
- **Path traversal hardening:** Active-workstream file content validated
- **Backward compatibility:** "flat mode" when no `workstreams/` directory exists

### Technical Debt

- **Worktree failure handling:** If `git worktree add` fails due to existing branch, tries timestamped fallback — may succeed silently with unexpected branch name
- **Workstream vs Workspace confusion:** Two related concepts may confuse users

---

## Feature 17: Backlog, Seeds & Threads

**Priority:** Secondary

### Description

Long-term idea management:
- **Backlog:** Parking lot using 999.x phase numbering
- **Seeds:** Forward-looking ideas with trigger conditions
- **Threads:** Persistent cross-session context

### Key Implementation Insights

**Backlog Numbering:** 999.x keeps backlog out of active phase sequence. No `Depends on:` field — unsequenced by definition.

**Seed File Location:** `.planning/seeds/SEED-NNN-slug.md`

**Seed Frontmatter:**
```yaml
---
id: SEED-{PADDED}
status: dormant
planted: {ISO date}
trigger_when: {$TRIGGER}
scope: {$SCOPE}
---
```

**Seed Trigger System:** Seeds have explicit trigger conditions that `/gsd:new-milestone` scans for. They don't just get lost in backlog.

**Thread File Location:** `.planning/threads/{slug}.md`

**Thread Status Lifecycle:** OPEN → IN PROGRESS → RESOLVED

### Notable Patterns

- **Breadcrumb collection:** Seeds proactively search codebase for related files, preserving context
- **Thread lightweight:** Deliberately avoids phase state, making session-independent
- **999.x numbering:** Stays out of any reasonable phase numbering scheme

### Technical Debt

- **Seed consumption not verifiable:** `/gsd:new-milestone` workflow not fully verified in reference repo
- **No seed aging:** Seeds remain dormant forever unless manually removed
- **Thread promotion manual:** No `/gsd:promote-thread` command

---

# Part III: Cross-Cutting Themes

## Architecture Patterns

### Thin Orchestrator Pattern
Every feature follows: orchestrator stays lean, spawns specialized agents, routes results. Orchestrator tracks state; agents do work. Never implements features directly.

### File-Based State as Source of Truth
All context lives in files that persist across sessions. Three layers:
1. **Artifact files** (PLAN.md, SUMMARY.md, UAT.md, VERIFICATION.md)
2. **Aggregate state** (STATE.md, ROADMAP.md)
3. **Git commits** — immutable, bisect-friendly audit trail

### Context Budget Enforcement
~50% context target per plan. 2-3 tasks maximum. Fresh 200k-token context per executor subagent. "Nyquist Rule" applied to agentic workflows.

### Wave-Based Parallelization
Independent plans run in parallel. Dependent plans wait for dependencies. Wave safety filter prevents deadlock.

## Security Patterns

### Defense-in-Depth
Multiple layers: path traversal has symlink resolution + containment check. Prompt injection has pattern matching + zero-width Unicode stripping + XML neutralization. Shell safety has null byte rejection + command substitution detection.

### Advisory-Only Prompt Guard
Hook cannot block — only surfaces warnings. Rationale: blocking would create false-positive deadlocks.

## Quality Patterns

### Three-Source Cross-Reference
Milestone audit validates requirements against VERIFICATION.md, SUMMARY frontmatter, and REQUIREMENTS.md traceability table independently. Catches orphaned requirements.

### Knowledge Base for Debugging
Persistent `.planning/debug/knowledge-base.md` enables learning across sessions. Resolved bugs become searchable hypothesis candidates.

### Gap Closure Pipeline
Gaps found during UAT flow through: diagnose (parallel debug) → plan (--gaps mode) → check → revise → execute (--gaps-only) → re-verify.

## Technical Debt Themes

1. **Pattern duplication:** Health checks, milestone audits, forensics all do validation differently
2. **No transaction safety:** Multiple-file writes without atomic guarantees
3. **Potential staleness:** Context files can drift from actual codebase state
4. **Windows platform issues:** MCP stdio deadlocks, stale task directories
5. **Incomplete test coverage:** security.cjs has no dedicated test file

## Key Files to Remember

| File | Purpose |
|------|---------|
| `workflows/execute-phase.md` | Main orchestrator for plan execution |
| `workflows/plan-phase.md` | Main orchestrator for plan creation |
| `workflows/verify-work.md` | UAT workflow with persistent state |
| `workflows/discuss-phase.md` | Decision capture before planning |
| `agents/gsd-planner.md` | Creates XML-structured plans |
| `agents/gsd-executor.md` | Executes plans with atomic commits |
| `agents/gsd-debugger.md` | Most methodologically rich agent |
| `bin/lib/security.cjs` | Path traversal, injection detection |
| `bin/lib/milestone.cjs` | Milestone CLI operations |
| `bin/lib/workstream.cjs` | Workstream management |
| `hooks/gsd-prompt-guard.js` | PreToolUse prompt injection guard |

---

## Summary

The GSD system is a sophisticated multi-agent orchestration framework built around file-based state management, wave-based parallel execution, and systematic verification. Its core strength is the tight integration between planning (context capture, requirement traceability, plan verification) and execution (fresh contexts, atomic commits, checkpoint-based human involvement).

The secondary features provide essential support: security hardening, session persistence, debugging methodology, milestone lifecycle management, and long-term idea management. Together they form a complete system for solo-developer productivity with Claude Code.

**Key architectural insight:** The system treats context as expensive and fragile — hence the 50% budget rule, fresh-context-per-plan pattern, and file-based persistence at every boundary. This is explicitly designed to fight "context rot" that plagues long-running AI-assisted development sessions.
