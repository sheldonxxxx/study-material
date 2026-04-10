# Feature Batch 5: Session Management, Debugging & Forensics, Milestone & Phase Management

**Research Date:** 2026-03-26
**Batch:** 5 of 6
**Features Covered:** Feature 13 (Session Management), Feature 14 (Debugging & Forensics), Feature 15 (Milestone & Phase Management)

---

## Feature 13: Session Management

### Feature Description
Pause/resume work with `HANDOFF.json` persistence, session reports, and health checks for `.planning/` directory integrity.

### Core Implementation Files

#### `get-shit-done/workflows/pause-work.md`
The primary session handoff workflow. Creates two parallel artifacts:

**Machine-readable:** `.planning/HANDOFF.json`
```json
{
  "version": "1.0",
  "timestamp": "{timestamp}",
  "phase": "{phase_number}",
  "phase_name": "{phase_name}",
  "plan": {current_plan_number},
  "task": {current_task_number},
  "status": "paused",
  "completed_tasks": [
    {"id": 1, "name": "{task_name}", "status": "done", "commit": "{short_hash}"},
    {"id": 2, "name": "{task_name}", "status": "done", "commit": "{short_hash}"},
    {"id": 3, "name": "{task_name}", "status": "in_progress", "progress": "{what_done}"}
  ],
  "remaining_tasks": [
    {"id": 4, "name": "{task_name}", "status": "not_started"},
    {"id": 5, "name": "{task_name}", "status": "not_started"}
  ],
  "blockers": [
    {"description": "{blocker}", "type": "technical|human_action|external", "workaround": "{if any}"}
  ],
  "next_action": "{specific first action when resuming}",
  "context_notes": "{mental state, approach, what you were thinking}"
}
```

**Human-readable:** `.planning/phases/XX-name/.continue-here.md`
Markdown handoff with sections: `<current_state>`, `<completed_work>`, `<remaining_work>`, `<decisions_made>`, `<blockers>`, `<context>`, `<next_action>`.

**Key flow:**
1. Detect current phase via `ls -lt .planning/phases/*/PLAN.md`
2. Gather: position, completed, remaining, decisions, blockers, pending human actions, background processes, uncommitted files
3. Check for placeholder content in SUMMARY files (flags false completions)
4. Write HANDOFF.json via `current-timestamp` tool
5. Write `.continue-here.md` with conversational context
6. Commit both as WIP: `wip: [phase-name] paused at task [X]/[Y]`

#### `get-shit-done/workflows/session-report.md`
Generates `.planning/reports/SESSION_REPORT.md` capturing:
- Git activity (last 24h)
- STATE.md position and blockers
- Work performed (phases touched, key outcomes, decisions)
- Files changed
- Estimated resource usage (commits, files changed, plans executed, subagents spawned)

**Notable:** Token/cost estimation is explicitly marked as unavailable — uses heuristics based on observable git activity. Each commit ~1 plan cycle, each plan file ~2,000-5,000 tokens.

#### `get-shit-done/workflows/health.md`
Validates `.planning/` directory integrity via `gsd-tools.cjs validate health`.

**Error code taxonomy:**

| Code | Severity | Description | Repairable |
|------|----------|-------------|------------|
| E001 | error | `.planning/` directory not found | No |
| E002 | error | PROJECT.md not found | No |
| E003 | error | ROADMAP.md not found | No |
| E004 | error | STATE.md not found | Yes |
| E005 | error | config.json parse error | Yes |
| W001 | warning | PROJECT.md missing required section | No |
| W002 | warning | STATE.md references invalid phase | No |
| W003 | warning | config.json not found | Yes |
| W004 | warning | config.json invalid field value | No |
| W005 | warning | Phase directory naming mismatch | No |
| W006 | warning | Phase in ROADMAP but no directory | No |
| W007 | warning | Phase on disk but not in ROADMAP | No |
| W008 | warning | `workflow.nyquist_validation` absent | Yes |
| W009 | warning | Phase has VALIDATION.md but no RESEARCH.md | No |
| I001 | info | Plan without SUMMARY (may be in progress) | No |

**Repair actions (via `--repair` flag):**
- `createConfig` — Create config.json with defaults
- `resetConfig` — Delete + recreate config.json
- `regenerateState` — Create STATE.md from ROADMAP structure
- `addNyquistKey` — Add `workflow.nyquist_validation: true` to config.json

**Windows-specific:** Detects and reports stale Claude Code task directories (older than 24h) in `~/.claude/tasks/` — leftover from crashed subagent sessions.

**Notable:** STATE.md content is explicitly NOT repairable (too risky to overwrite session history). Only structural absence is repairable.

### Flow Trace

```
/gsd:pause-work
  └─> detect phase (ls -lt PLAN.md)
  └─> gather state from STATE.md, git log, SUMMARY files
  └─> write .planning/HANDOFF.json (machine-readable)
  └─> write .planning/phases/XX-name/.continue-here.md (human-readable)
  └─> git commit WIP

/resume-work (external)
  └─> reads HANDOFF.json
  └─> restores state
  └─> continues from next_action

/gsd:session-report
  └─> gather from STATE.md, git log, ROADMAP.md
  └─> write .planning/reports/SESSION_REPORT.md (dated, never overwrites)

/gsd:health
  └─> gsd-tools.cjs validate health [--repair]
  └─> display errors/warnings/info with fix instructions
  └─> offer repair if --repair not used
```

### Notable Implementation Details

**Good:**
- Dual-format handoff (JSON + Markdown) serves both machine automation and human comprehension
- `pause-work` explicitly asks user for clarifications via conversational questions before writing handoff — prevents assumptions
- Checks for placeholder content in summaries as false-completion detection
- Health checks differentiate error/warning/info severity and repairable/non-repairable
- Human actions pending are explicitly categorized with `type` field (technical/human_action/external)

**Concerns/Shortcuts:**
- `pause-work` relies on `ls -lt .planning/phases/*/PLAN.md` to find active phase — fragile if PLAN.md exists but phase is complete
- `session-report` token estimation is acknowledged as crude heuristic — no actual API-level instrumentation
- No automatic cleanup of old session reports; if reports accumulate, the `ls` command would show them but there's no rotation policy
- Resume-work workflow file not found in the expected location — appears to be delegated entirely to a command-level implementation

---

## Feature 14: Debugging & Forensics

### Feature Description
Systematic debugging with persistent state (`/gsd:debug`) and post-mortem investigation of failed workflow runs (`/gsd:forensics`).

### Core Implementation Files

#### `agents/gsd-debugger.md` (1,374 lines)
The primary debugging agent. Loaded with extensive methodology.

**Key philosophy:** User = Reporter (knows symptoms), Claude = Investigator (finds cause). The debugger is explicitly instructed to treat its own code as foreign when debugging code it wrote — fighting the "my code is obviously correct" bias.

**File-based session persistence:** `.planning/debug/{slug}.md`
```markdown
---
status: gathering | investigating | fixing | verifying | awaiting_human_verify | resolved
trigger: "[verbatim user input]"
created: [ISO timestamp]
updated: [ISO timestamp]
---

## Current Focus   (OVERWRITE on each update)
hypothesis: [current theory]
test: [how testing it]
expecting: [what result means]
next_action: [immediate next step]

## Symptoms        (IMMUTABLE after gathering)
expected: [what should happen]
actual: [what actually happens]
errors: [error messages]
reproduction: [how to trigger]
started: [when broke / always broken]

## Eliminated      (APPEND only)
- hypothesis: [theory that was wrong]
  evidence: [what disproved it]
  timestamp: [when eliminated]

## Evidence        (APPEND only)
- timestamp: [when found]
  checked: [what examined]
  found: [what observed]
  implication: [what this means]

## Resolution      (OVERWRITE as understanding evolves)
root_cause: [empty until found]
fix: [empty until applied]
verification: [empty until verified]
files_changed: []
```

**Status state machine:**
```
gathering -> investigating -> fixing -> verifying -> awaiting_human_verify -> resolved
                  ^            |           |                 |
                  |____________|___________|_________________|
                  (if verification fails or user reports issue)
```

**Knowledge base:** `.planning/debug/knowledge-base.md`
Append-only record of resolved sessions. New sessions check for keyword overlap at investigation start. Match = hypothesis candidate (not certainty).

```markdown
## {slug} — {one-line description}
- **Date:** {ISO date}
- **Error patterns:** {comma-separated keywords from symptoms}
- **Root cause:** {from Resolution}
- **Fix:** {from Resolution}
- **Files changed:** {from Resolution}
---
```

**Mode flags:**
- `symptoms_prefilled: true` — skip gathering, go straight to investigation
- `goal: find_root_cause_only` — diagnose but don't fix; return diagnosis and exit
- `goal: find_and_fix` (default) — full cycle

**Hypothesis testing discipline:**
- Falsifiability required: specific, testable claims (not "state is wrong")
- One hypothesis at a time
- Evidence quality rated: strong (repeatable, unambiguous) vs weak (hearsay, non-repeatable)
- Binary search / divide-and-conquer for large codebases

**Investigation techniques catalogued:**
| Technique | When to Use |
|-----------|-------------|
| Binary search | Large codebase, long execution path |
| Rubber duck | Stuck, confused, mental model mismatch |
| Minimal reproduction | Complex system, unclear failure point |
| Working backwards | Know desired output, not current |
| Differential debugging | Used to work, now doesn't |
| Comment out everything | Many possible interactions |
| Git bisect | Worked in past, broke at unknown commit |
| Follow the indirection | Paths/URLs/keys constructed from variables |

**Verification patterns:**
- Reproduction verification: exact steps before and after
- Regression testing checklist
- Stability testing (repeated runs for intermittent bugs)
- Red flag phrases: "seems to work", "I think it's fixed"

#### `get-shit-done/workflows/forensics.md`
Post-mortem investigation workflow. **Explicitly read-only** — only writes the forensic report.

**Evidence gathering:**
1. Git history: commits, timestamps, most-edited files, uncommitted changes
2. Planning state: STATE.md, ROADMAP.md, config.json
3. Phase artifacts: PLAN.md, SUMMARY.md, VERIFICATION.md, CONTEXT.md, RESEARCH.md
4. Session reports: `.planning/reports/SESSION_REPORT.md`
5. Git worktree state: orphaned worktrees from crashed agents

**Anomaly detection patterns:**

**Stuck Loop Detection:**
- Signal: Same file in 3+ consecutive commits within short time window
- Confidence HIGH if commit messages are similar ("fix:", "fix:", "fix:")
- Confidence MEDIUM if file appears frequently but messages vary

**Missing Artifact Detection:**
- Phase appears complete but lacks expected artifacts
- PLAN.md missing = planning skipped
- SUMMARY.md missing = phase not properly closed
- VERIFICATION.md missing = quality check skipped

**Abandoned Work Detection:**
- Large gap between last commit and current time
- STATE.md shows mid-execution
- >2 hours with uncommitted changes = potential abandonment

**Crash/Interruption Detection:**
- Uncommitted changes + STATE.md mid-execution + orphaned worktrees

**Scope Drift Detection:**
- Recent commits touch files outside current phase's expected scope (read from PLAN.md)

**Test Regression Detection:**
- Commit messages with "fix test", "revert", "broken", "regression", "fail"

**Report output:** `.planning/forensics/report-$(date +%Y%m%d-%H%M%S).md`
Structured with Evidence Summary, Anomaly table, Root Cause Hypothesis, Recommended Actions.

**Notable:** Includes GitHub issue creation via `gh issue create` with bug label detection.

### Flow Trace

```
/gsd:debug [issue description]
  ├─> check for active sessions (ls .planning/debug/*.md)
  ├─> create debug file immediately (status: gathering)
  ├─> gather symptoms via questioning
  ├─> Phase 0: check knowledge-base for pattern match
  ├─> Phase 1: gather evidence (read files, run tests)
  ├─> Phase 2: form falsifiable hypothesis
  ├─> Phase 3: test one at a time
  ├─> Phase 4: evaluate (confirmed → fix_and_verify, eliminated → new hypothesis)
  ├─> fix_and_verify: implement minimal fix, verify against symptoms
  ├─> request_human_verification: checkpoint
  ├─> archive_session: move to resolved/, update knowledge base
  └─> commit fix + knowledge base update

/gsd:forensics [problem description]
  ├─> gather evidence (git history, planning state, phase artifacts, session reports)
  ├─> detect anomalies (stuck loops, missing artifacts, abandoned work, scope drift)
  ├─> generate forensic report to .planning/forensics/report-{timestamp}.md
  ├─> offer deeper investigation
  ├─> offer GitHub issue creation
  └─> update STATE.md
```

### Notable Implementation Details

**Excellent:**
- `gsd-debugger.md` is the most methodologically rich agent in the codebase — 1,374 lines of systematic investigation discipline
- Knowledge base protocol enables learning across sessions — resolved bugs become searchable hypothesis candidates
- File-based persistence with immutable Symptoms section ensures evidence isn't revised under pressure
- Status machine with explicit transition rules prevents ad-hoc debugging
- Red flag phrases list is unusually honest — "I think it's fixed" and "seems to work" are flagged as trust-building failures
- `goal: find_root_cause_only` mode allows forensics to use the debugger without full fix cycle
- Differential debugging explicitly addresses "works in dev, fails in prod" class of problems

**Shortcuts/Concerns:**
- Knowledge base matching is keyword overlap (2+ tokens), not semantic — could surface false positives
- The `Follow the indirection` technique mentions "AH!" moment examples but the discipline requires the AI to actually follow the indirection — no automated enforcement
- No integration with actual debugging tools (Node debugger, browser DevTools, etc.) — purely code reading + logging
- `forensics.md` anomaly detection is pattern-based on git history — could miss issues that don't leave obvious git traces
- Orphaned worktree detection via `git worktree list` is fragile — depends on user not having legitimate worktrees

---

## Feature 15: Milestone & Phase Management

### Feature Description
Complete milestone lifecycle (new, audit, complete) and phase management (add, insert, remove, validate) with gap closure planning.

### Core Implementation Files

#### `get-shit-done/workflows/complete-milestone.md`
Marks a shipped version complete. The most complex workflow in the system (700+ lines).

**Key steps:**
1. **verify_readiness** — `gsd-tools.cjs roadmap analyze` for comprehensive phase status. Requirements completion check against traceability table. User confirms or adjusts scope.
2. **gather_stats** — Git log, diff stat, LOC counting, timeline from first to last commit
3. **extract_accomplishments** — `gsd-tools.cjs summary-extract` for each phase's one-liner
4. **evolve_project_full_review** — Full PROJECT.md evolution at milestone boundary:
   - "What This Is" accuracy check
   - Core Value verification
   - Requirements audit (Validated ← Active, Active ← new)
   - Out of Scope audit
   - Context update (LOC, tech stack, user feedback themes)
   - Key Decisions audit (add milestone decisions with outcomes, mark Good/Revisit/Pending)
   - Constraints check
5. **archive_milestone** — Delegates to `gsd-tools.cjs milestone complete`:
   - Creates `.planning/milestones/` directory
   - Archives ROADMAP.md → `milestones/v[X.Y]-ROADMAP.md`
   - Archives REQUIREMENTS.md → `milestones/v[X.Y]-REQUIREMENTS.md`
   - Moves audit file if exists
   - Creates/appends MILESTONES.md entry
   - Updates STATE.md
6. **reorganize_roadmap_and_delete_originals** — Groups completed phases into `<details>` collapsible, deletes original ROADMAP.md and REQUIREMENTS.md
7. **handle_branches** — Branch strategy handling (none/phase/milestone), squash merge vs merge with history
8. **git_tag** — `git tag -a v[X.Y]` with multi-line tag message
9. **write_retrospective** — Append to RETROSPECTIVE.md, update cross-milestone trends

**Delegation pattern:** `gsd-tools.cjs milestone complete` is the CLI workhorse that handles file I/O. The workflow orchestrates and provides judgment calls.

#### `get-shit-done/workflows/audit-milestone.md`
Validates milestone achieved its definition of done by aggregating phase verifications.

**3-source cross-reference requirement coverage check:**

| VERIFICATION.md Status | SUMMARY Frontmatter | REQUIREMENTS.md | Final Status |
|-----------------------|---------------------|----------------|--------------|
| passed | listed | `[x]` | **satisfied** |
| passed | listed | `[ ]` | **satisfied** (update checkbox) |
| passed | missing | any | **partial** |
| gaps_found | any | any | **unsatisfied** |
| missing | listed | any | **partial** |
| missing | missing | any | **unsatisfied** |

**Orphan detection:** Requirements in traceability table but absent from ALL phase VERIFICATION.md files are flagged as orphaned (treated as unsatisfied).

**Nyquist compliance discovery:** Checks for `*-VALIDATION.md` in each phase directory. Classifies per phase: COMPLIANT / PARTIAL / MISSING.

**Output:** `.planning/v{version}-MILESTONE-AUDIT.md` with structured YAML frontmatter:
```yaml
---
milestone: {version}
audited: {timestamp}
status: passed | gaps_found | tech_debt
scores:
  requirements: N/M
  phases: N/M
  integration: N/M
  flows: N/M
gaps:
  requirements:
    - id: "{REQ-ID}"
      status: "unsatisfied | partial | orphaned"
      phase: "{assigned phase}"
      claimed_by_plans: [...]
      completed_by_plans: [...]
      verification_status: "passed | gaps_found | missing | orphaned"
      evidence: "{specific evidence}"
  integration: [...]
  flows: [...]
tech_debt:
  - phase: 01-auth
    items:
      - "TODO: add rate limiting"
---
```

**Integration checker spawns:** Uses `gsd-integration-checker` subagent for cross-phase wiring verification.

#### `get-shit-done/workflows/plan-milestone-gaps.md`
Creates all phases necessary to close gaps from `/gsd:audit-milestone`. Reads MILESTONE-AUDIT.md, groups gaps into logical phases.

**Gap-to-task derivation:**
- Requirement gap → tasks mapping concrete missing implementation
- Integration gap → tasks for missing cross-phase connections
- Flow gap → tasks for broken E2E user journeys

**Clustering rules:**
- Same affected phase → combine into one fix phase
- Same subsystem → combine
- Dependency order (fix stubs before wiring)
- 2-4 tasks per phase

**Key behavior:** Resets `[x]` → `[ ]` for unsatisfied requirements in REQUIREMENTS.md. Updates traceability table phase assignments to new gap closure phases.

#### `get-shit-done/workflows/add-phase.md`
Adds new integer phase to end of current milestone. Single-purpose, delegates to `gsd-tools.cjs phase add`.

```bash
RESULT=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" phase add "${description}")
```

CLI handles: finding highest existing phase number, generating slug, creating directory, inserting into ROADMAP.md.

#### `get-shit-done/workflows/new-milestone.md`
Starts a new milestone cycle. Orchestrates research → requirements → roadmap.

**Parallel research:** Spawns 4 `gsd-project-researcher` agents for Stack, Features, Architecture, Pitfalls dimensions. Then `gsd-research-synthesizer` to create SUMMARY.md.

**Requirement gathering:** Scoped per category (table stakes vs differentiators). REQ-ID format: `[CATEGORY]-[NUMBER]`.

**Roadmap creation:** Spawns `gsd-roadmapper` agent to derive phases from requirements, map every REQ-ID to exactly one phase, derive 2-5 success criteria per phase.

**Phase numbering modes:**
- Default: continues from previous milestone's last phase (v1.0 ended at phase 5 → v1.1 starts at phase 6)
- `--reset-phase-numbers` flag: restarts at Phase 1, archives old phase directories first

#### `get-shit-done/bin/lib/milestone.cjs`
CLI implementation of milestone operations.

**`cmdMilestoneComplete`:**
- Uses `getMilestonePhaseFilter(cwd)` to scope stats to current milestone only (same filter used by `cmdPhasesList`)
- Counts phases, plans, tasks from phase directories
- Extracts one-liners from SUMMARY.md frontmatter via `extractFrontmatter` + `extractOneLinerFromBody`
- Task counting: prefers `**Tasks:** N` from Performance section, falls back to counting `<task XML tags` or `## Task N markdown headers`
- Archives: ROADMAP.md, REQUIREMENTS.md (with archive header), audit file
- MILESTONES.md: inserts entry after header (reverse chronological = newest first)
- Updates STATE.md via `stateReplaceFieldWithFallback` (handles both `**Field:**` and `Field:` formats)

**`cmdRequirementsMarkComplete`:**
- Accepts comma-separated, space-separated, or bracket-wrapped IDs
- Updates checkbox: `- [ ] **REQ-ID**` → `- [x] **REQ-ID**`
- Updates traceability table: `| REQ-ID | Phase N | Pending |` → `| REQ-ID | Phase N | Complete |`
- Detects already-complete vs not-found vs updated

#### `get-shit-done/bin/lib/phase.cjs`
Phase CRUD and query operations.

**`cmdPhasesList`:**
- Sorts numerically (handles integers, decimals, letter-suffix, hybrids)
- Supports `--type plans|summaries` for file listing
- Supports `--include-archived` for archived phase directories
- `phase` filter returns single matching directory

**`cmdPhaseNextDecimal`:**
- Calculates next decimal phase (e.g., base 3 → next 3.1)
- Used by gap closure phases that insert between existing phases

### Flow Trace

```
/gsd:add-phase <description>
  └─> gsd-tools.cjs phase add
  └─> update STATE.md roadmap evolution
  └─> commit

/gsd:new-milestone [--reset-phase-numbers] [name]
  ├─> load PROJECT.md, MILESTONES.md, STATE.md
  ├─> gather milestone goals (MILESTONE-CONTEXT.md or conversation)
  ├─> confirm version (v1.0 → v1.1 or v2.0)
  ├─> optional: parallel research (4 agents)
  ├─> define requirements with REQ-IDs
  ├─> spawn gsd-roadmapper to create ROADMAP.md
  ├─> commit all artifacts
  └─> user: /gsd:discuss-phase [N]

/gsd:audit-milestone
  ├─> init milestone-op context
  ├─> read all phase VERIFICATION.md files
  ├─> spawn gsd-integration-checker for cross-phase wiring
  ├─> 3-source cross-reference (VERIFICATION + SUMMARY frontmatter + traceability)
  ├─> detect orphaned requirements
  ├─> aggregate tech debt
  ├─> check Nyquist compliance
  └─> write v{version}-MILESTONE-AUDIT.md
  └─> route by status (passed / gaps_found / tech_debt)

/gsd:plan-milestone-gaps
  ├─> find most recent MILESTONE-AUDIT.md
  ├─> prioritize gaps (must/should/nice from REQUIREMENTS.md priority)
  ├─> cluster into phases (2-4 tasks each)
  ├─> user confirms
  ├─> update ROADMAP.md
  ├─> update REQUIREMENTS.md traceability (reset [x] → [ ] for unsatisfied)
  ├─> create phase directories
  └─> commit
  └─> user: /gsd:plan-phase [N]

/gsd:complete-milestone v[X.Y] --name "[Name]"
  ├─> verify readiness (roadmap analyze, requirements check)
  ├─> gather stats (git log, diff, LOC)
  ├─> extract accomplishments from SUMMARY files
  ├─> full PROJECT.md evolution review
  ├─> gsd-tools milestone complete (archive ROADMAP, REQUIREMENTS, update MILESTONES.md, STATE.md)
  ├─> reorganize ROADMAP.md (milestone grouping, delete originals)
  ├─> handle branches (merge strategy)
  ├─> git tag v[X.Y]
  ├─> write RETROSPECTIVE.md
  └─> commit milestone completion
  └─> user: /gsd:new-milestone
```

### Notable Implementation Details

**Excellent:**
- Milestone completion properly separates CLI work (file I/O, `milestone.cjs`) from judgment work (PROJECT.md evolution, RETROSPECTIVE, branch handling) — the workflow provides the thinking, the CLI does the moving
- 3-source cross-reference for requirements coverage is rigorous — catches orphaned requirements (in traceability but never verified)
- `getMilestonePhaseFilter` is shared between `cmdPhasesList` and `cmdMilestoneComplete` — ensures consistent milestone scoping
- Task counting in `cmdMilestoneComplete` has fallback chain: `**Tasks:** N` field → XML `<task` tags → `## Task N` markdown headers — handles all common formats
- Phase numbering supports decimal phases (e.g., 5.1) for gap closure phases inserted between existing phases
- `--reset-phase-numbers` safety: archives old phase directories before starting new numbering to prevent collision

**Shortcuts/Concerns:**
- `complete-milestone.md` is extremely long (700+ lines) with deep nesting — makes the workflow hard to follow and modify
- The RETROSPECTIVE.md cross-milestone trends update assumes a specific table format exists — no schema enforcement
- Milestone archival deletes original ROADMAP.md and REQUIREMENTS.md after archiving — if archival fails mid-way, could lose state
- `plan-milestone-gaps` groups gaps into phases using rules ("same subsystem → combine") but the grouping is ultimately subjective — no automated enforcement
- Nyquist compliance discovery in `audit-milestone` is explicitly "discovery only" — never auto-calls `/gsd:validate-phase`, leaving a gap between detecting partial compliance and actually fixing it
- Health check (Feature 13) and milestone audit (Feature 15) both do validation but via different mechanisms — no shared validation infrastructure

---

## Cross-Cutting Observations

### Technical Debt Patterns

1. **File-based state without transactions:** All three features write multiple files but there's no atomic multi-file write. A crash between writes could leave inconsistent state. The `pause-work` commit is the only transactional guarantee.

2. **Pattern duplication across features:** Health checks (Feature 13), milestone audits (Feature 15), and forensics (Feature 14) all do validation but share no infrastructure. Health uses error codes E/W/I, audits use YAML frontmatter with status/score/gaps/tech_debt, forensics uses pattern detection on git history.

3. **Missing `resume-work.md`:** The session management feature pair (pause/resume) shows pause-work implemented as a full workflow, but resume-work was not found as a corresponding workflow file. Likely implemented as a lighter command that just reads HANDOFF.json and restores state.

### Architecture Strengths

1. **Delegation pattern:** Complex CLI operations (milestone complete, phase add, validate health) are handled by `gsd-tools.cjs` library modules. Workflows orchestrate judgment calls while libs handle file I/O.

2. **Knowledge base for debugging:** The persistent knowledge base (`.planning/debug/knowledge-base.md`) enables learning across sessions — a sophisticated pattern for a workflow system.

3. **Three-source cross-reference:** The milestone audit's requirement coverage check independently validates against VERIFICATION.md, SUMMARY frontmatter, and REQUIREMENTS.md traceability table. Catches orphaned requirements that pass one or two sources but not all three.

4. **Separation of read-only investigation vs. modification:** `forensics.md` explicitly writes only the report file — all other files are read-only. This is a good discipline that other workflows could learn from.

### Inter-Feature Dependencies

- Feature 13 (Session Management) is foundational — pause-work writes the HANDOFF.json that resume-work reads. Health checks validate the `.planning/` structure that all other features depend on.
- Feature 14 (Debugging) is orthogonal — used when other features encounter problems. The `goal: find_root_cause_only` mode explicitly supports gap closure planning.
- Feature 15 (Milestone & Phase Management) depends on Feature 13's health checks for `.planning/` integrity before completing milestones.
- All three features write to `.planning/` — potential conflict if multiple run simultaneously (though the CLI-heavy architecture reduces this).
