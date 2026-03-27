# Features Batch 1: Project Initialization, Phase Discussion, Phase Planning

**Research Date:** 2026-03-26
**Batch:** 1 of 6
**Features:** Project Initialization, Phase Discussion, Phase Planning

---

## Feature 1: Project Initialization

### Description
Single-command project setup that transforms ideas into structured plans via guided questions, research, and roadmap generation. This is the most leveraged moment in any project - deep questioning here means better plans, better execution, better outcomes.

### Core Implementation Files

| File | Purpose |
|------|---------|
| `get-shit-done/workflows/new-project.md` | Main orchestrator workflow |
| `agents/gsd-project-researcher.md` | Spawned researcher for domain ecosystem |
| `agents/gsd-roadmapper.md` | Spawned roadmap creator |
| `agents/gsd-research-synthesizer.md` | Synthesizes parallel research into SUMMARY |

### Flow Trace (End-to-End)

```
User invokes /gsd:new-project
        ↓
1. Setup (init tool, parse config)
        ↓
2. Brownfield detection
   - If existing code without map → offer /gsd:map-codebase first
   - If greenfield → continue
        ↓
3. Deep Questioning (greenfield) OR
   Auto-mode config extraction (if --auto with PRD)
        ↓
4. Write PROJECT.md
   - Core value, constraints, "What This Is"
   - Evolution section for lifecycle management
        ↓
5. Workflow Preferences
   - Mode (YOLO/Interactive)
   - Granularity (Coarse/Standard/Fine)
   - Parallelization (Parallel/Sequential)
   - Git tracking preference
   - Research/Plan-Check/Verifier toggles
   - Model profile (Balanced/Quality/Budget/Inherit)
        ↓
6. Research Decision
   - If "Research first" → spawn 4 parallel gsd-project-researcher agents:
     * Stack research
     * Features research
     * Architecture research
     * Pitfalls research
   - Then spawn gsd-research-synthesizer to create SUMMARY.md
        ↓
7. Define Requirements
   - Present features by category (from research or conversation)
   - Scope each category: v1 / v2 / Out of Scope
   - Generate REQUIREMENTS.md with REQ-IDs
        ↓
8. Create Roadmap
   - Spawn gsd-roadmapper with all context
   - 100% requirement coverage validation
   - Success criteria per phase (goal-backward)
   - UI hint detection for frontend phases
        ↓
9. Done
   - Project initialized with all artifacts committed
   - Auto-advance to discuss-phase 1 (if --auto)
```

### Key Code Snippets

**Research Agent Spawning Pattern:**
```markdown
Task(prompt="<research_type>
Project Research — Stack dimension for [domain].
</research_type>

<milestone_context>
[greenfield OR subsequent]
</milestone_context>

<question>
What's the standard 2025 stack for [domain]?
</question>

<files_to_read>
- {project_path}
</files_to_read>

${AGENT_SKILLS_RESEARCHER}

<downstream_consumer>
Your STACK.md feeds into roadmap creation. Be prescriptive:
- Specific libraries with versions
- Clear rationale for each choice
- What NOT to use and why
</downstream_consumer>
...
", subagent_type="gsd-project-researcher", model="{researcher_model}", description="Stack research")
```

**Config Creation Pattern:**
```bash
node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" config-new-project \
  '{"mode":"[yolo|interactive]","granularity":"[selected]",\
    "parallelization":true|false,"commit_docs":true|false,\
    "model_profile":"quality|balanced|budget|inherit",\
    "workflow":{"research":true|false,"plan_check":true|false,\
    "verifier":true|false,"nyquist_validation":[false if coarse]}}'
```

### Notable Implementation Details

**GOOD: Auto-mode PRD Express Path**
When `--auto @prd.md` is provided, the workflow extracts requirements directly from the document without interactive questioning. This is clever for CI/CD pipelines and experienced users.

**GOOD: Brownfield Detection**
The `init` tool detects existing code via `has_package_file`, `is_brownfield` flags. If codebase exists without a map, it offers to run `/gsd:map-codebase` first. This prevents retrofitting GSD onto existing projects without understanding them.

**GOOD: Sub-Repo Detection**
```bash
find . -maxdepth 1 -type d -not -name ".*" -not -name "node_modules" -exec test -d "{}/.git" \; -print
```
Detects multi-repo workspaces and configures `planning.sub_repos` accordingly.

**GOOD: Research Agent Skills**
Uses `agent-skills` tool to inject skill context into spawned researchers, ensuring research follows project conventions.

**GOOD: Milestone Context Differentiation**
Greenfield vs subsequent milestone determines research scope - researchers know whether to recommend from scratch or how to add to existing systems.

**TECHNICAL DEBT: Config Initialization Order**
Config is created BEFORE knowing if it's greenfield or brownfield. The `config-new-project` tool auto-populates defaults rather than accepting full explicit config. If user declines research later, model_profile set in config may not match what they would have chosen interactively.

**TECHNICAL DEBT: No Rollback on Research Failure**
If 4 parallel researchers spawn but one fails, the synthesizer still runs with partial results. No explicit error handling for partial research completion.

**SHORTCUT: Config Defaults**
```javascript
// From config.cjs pattern - defaults are applied automatically
// If key absent: granularity → "standard", parallelization → true
```
Users can skip settings if defaults match their preferences.

---

## Feature 2: Phase Discussion

### Description
Captures implementation decisions and preferences before planning, producing CONTEXT.md that locks user decisions and feeds researcher/planner. A thinking partnership that captures decisions, not implementation.

### Core Implementation Files

| File | Purpose |
|------|---------|
| `get-shit-done/workflows/discuss-phase.md` | Main workflow orchestrator |
| `agents/gsd-assumptions-analyzer.md` | Analyzes codebase for phase (assumptions mode) |

### Flow Trace (End-to-End)

```
User invokes /gsd:discuss-phase {phase}
        ↓
1. Initialize
   - init plan-phase tool
   - Parse phase_found, phase_dir, has_context, has_plans
        ↓
2. Check Existing Context
   - If CONTEXT.md exists → Update/View/Skip options
   - If plans exist → Continue and replan / View / Cancel
        ↓
3. Load Prior Context
   - Read PROJECT.md, REQUIREMENTS.md, STATE.md
   - Read all prior phase CONTEXT.md files
   - Build <prior_decisions> structure
        ↓
4. Cross-Reference TODOs
   - Match pending todos to phase scope
   - Offer to fold into scope or defer
        ↓
5. Scout Codebase
   - Check for existing codebase maps
   - If none: targeted grep for phase-related files
   - Build <codebase_context> for discussion grounding
        ↓
6. Analyze Phase
   - Identify domain boundary
   - Generate phase-specific gray areas
   - Apply advisor mode if enabled (USER-PROFILE.md exists)
   - Accumulate canonical refs
        ↓
7. Present Gray Areas
   - User selects which areas to discuss
   - Prior decisions annotated
   - Code context annotations
        ↓
8. Advisor Research (if ADVISOR_MODE)
   - Spawn parallel research for each selected area
   - Synthesize comparison tables
   - Present with rationale
        ↓
9. Discuss Areas
   - For each area:
     * Present question with options
     * Capture answer
     * If "Other" → receive free text
     * Follow-up if needed
   - Track discussion log
        ↓
10. Write Context
    - CONTEXT.md with decisions
    - DISCUSSION-LOG.md for audit trail
    - Canonical refs section (MANDATORY)
        ↓
11. Git Commit
    - Commit both files
    - Update STATE.md
        ↓
12. Auto-Advance
    - If --auto → launch /gsd:plan-phase
```

### Key Code Snippets

**Gray Area Identification Logic:**
```markdown
3. **Gray areas by category** — For each relevant category (UI, UX, Behavior,
   Empty States, Content), identify 1-2 specific ambiguities that would change
   implementation. **Annotate with code context where relevant**

4. **Skip assessment** — If no meaningful gray areas exist (pure infrastructure,
   clear-cut implementation, or all already decided in prior phases), the phase
   may not need discussion.
```

**Scope Creep Guardrail:**
```markdown
**The heuristic:** Does this clarify how we implement what's already in the
phase, or does it add a new capability that could be its own phase?

**When user suggests scope creep:**
"[Feature X] would be a new capability — that's its own phase.
Want me to note it for the roadmap backlog?

For now, let's focus on [phase domain]."
```

**Answer Validation:**
```markdown
**IMPORTANT: Answer validation** — After every AskUserQuestion call, check if
the response is empty or whitespace-only. If so:
1. Retry the question once with the same parameters
2. If still empty, present the options as a plain-text numbered list
```

**Advisor Mode Model Resolution:**
```bash
ADVISOR_MODEL=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" \
  resolve-model gsd-advisor-researcher --raw)
```

### Notable Implementation Details

**GOOD: Prior Decisions Carry-Forward**
Builds `<prior_decisions>` from earlier phases so decisions made in Phase 2 don't get re-asked in Phase 5. Annotates gray areas with "You chose X in Phase N".

**GOOD: Advisor Mode (User Profiling)**
If `USER-PROFILE.md` exists and `vendor_philosophy` is set, calibrates advisor research depth:
- conservative/thorough-evaluator → full_maturity (3-5 options)
- opinionated → minimal_decisive (1-2 options)
- pragmatic-fast → standard (2-4 options)

**GOOD: Canonical Refs Accumulation**
The workflow actively accumulates canonical references throughout discussion:
1. Copy from ROADMAP.md initial refs
2. Check REQUIREMENTS.md and PROJECT.md
3. When user says "read X", "check Y" → add immediately
4. When scout finds code references → add

**GOOD: Text Mode Support**
```bash
workflow.text_mode: true  # from config
--text flag               # from arguments
```
When active, replaces ALL AskUserQuestion with numbered lists. Required for Claude Code remote sessions (`/rc` mode) where TUI menus don't work.

**GOOD: Batch Mode**
```bash
--batch, --batch=N, --batch N
```
Allows grouping 2-5 questions per turn for faster discussion.

**GOOD: DISCUSSION-LOG.md Audit Trail**
Full Q&A preserved separately from CONTEXT.md for compliance reviews, not consumed by downstream agents.

**TECHNICAL DEBT: Deep Agent Nesting Warning**
```markdown
**IMPORTANT:** Do NOT invoke discuss-phase as a nested Skill/Task call —
AskUserQuestion does not work correctly in nested subcontexts (#1009).
```
When user says "Run discuss-phase first", workflow must EXIT and let user re-invoke. Cannot chain nested.

**SHORTCUT: Auto Mode Defaults**
```markdown
**If `--auto`:** Auto-select ALL gray areas without asking the user.
Log each auto-selected choice inline.
For each discussion question, choose the recommended option.
```
No actual discussion happens - just logs what would have been chosen.

**SHORTCUT: PRD Express Path in plan-phase**
Rather than discuss, user can provide `--prd path/to/prd.md` to skip discussion and go straight to planning. All PRD requirements become locked decisions.

---

## Feature 3: Phase Planning

### Description
Multi-agent planning with research synthesis, XML-structured plan creation, and automated verification against requirements. Creates 2-3 atomic task plans per phase. Orchestrates gsd-phase-researcher, gsd-planner, and gsd-plan-checker with revision loop (max 3 iterations).

### Core Implementation Files

| File | Purpose |
|------|---------|
| `get-shit-done/workflows/plan-phase.md` | Main orchestrator |
| `agents/gsd-phase-researcher.md` | Phase-level technical research |
| `agents/gsd-planner.md` | Creates PLAN.md files |
| `agents/gsd-plan-checker.md` | Verifies plans before execution |

### Flow Trace (End-to-End)

```
User invokes /gsd:plan-phase {phase}
        ↓
1. Initialize
   - init plan-phase tool
   - Parse all flags: --research, --skip-research, --gaps, --skip-verify, --prd, --reviews, --text
        ↓
2. Validate Phase
   - Phase exists in ROADMAP.md?
   - Create directory if needed
        ↓
3. Handle PRD Express Path (if --prd)
   - Generate CONTEXT.md from PRD
   - All PRD requirements become locked decisions
   - Skip to step 5
        ↓
4. Load CONTEXT.md
   - If missing → offer Continue/Gather Context/Run discuss-phase first
   - If discuss-phase needed → EXIT (no nested AskUserQuestion)
        ↓
5. Handle Research
   - If --skip-research or --gaps or --reviews → skip
   - If research exists and no --research flag → use existing
   - If research needed → spawn gsd-phase-researcher
   - Create VALIDATION.md (Nyquist) if enabled
        ↓
6. UI Design Contract Gate
   - Check if phase has UI indicators
   - If yes and no UI-SPEC.md → offer /gsd:ui-phase first
        ↓
7. Check Existing Plans
   - If exist → Add/View/Replan options
   - If --reviews flag → go straight to replan
        ↓
8. Spawn gsd-planner Agent
   - Goal-backward plan creation
   - 2-3 tasks per plan
   - Wave assignment for parallelization
   - must_haves derivation
        ↓
9. Handle Planner Return
   - PLANNING COMPLETE → continue to step 10
   - CHECKPOINT REACHED → present to user
   - PLANNING INCONCLUSIVE → offer Add context/Retry/Manual
        ↓
10. Spawn gsd-plan-checker (if not --skip-verify)
    - 10 verification dimensions
    - Goal-backward verification
    - Context compliance check
        ↓
11. Handle Checker Return
    - ISSUES FOUND → revision loop (max 3 iterations)
    - VERIFICATION PASSED → continue
        ↓
12. Revision Loop
    - Send checker issues back to planner
    - Planner makes targeted fixes
    - Re-check with checker
        ↓
13. Requirements Coverage Gate
    - All REQ-IDs from ROADMAP must be covered by plans
    - If gaps → Re-plan/Move to next phase/Proceed anyway
        ↓
14. Present Final Status
    - Auto-advance to /gsd:execute-phase (if --auto)
    - Otherwise → show next steps
```

### Key Code Snippets

**Planner Agent Prompt Structure:**
```markdown
<planning_context>
**Phase:** {phase_number}
**Mode:** {standard | gap_closure | reviews}

<files_to_read>
- {state_path}
- {roadmap_path}
- {requirements_path}
- {context_path}
- {research_path}
- {verification_path} (if --gaps)
- {uat_path} (if --gaps)
- {reviews_path} (if --reviews)
- {UI_SPEC_PATH} (if exists)
</files_to_read>

${AGENT_SKILLS_PLANNER}

**Phase requirement IDs:** {phase_req_ids}
</planning_context>

<downstream_consumer>
Output consumed by /gsd:execute-phase. Plans need:
- Frontmatter (wave, depends_on, files_modified, autonomous)
- Tasks in XML format with read_first and acceptance_criteria
- Verification criteria
- must_haves for goal-backward verification
</downstream_consumer>

<deep_work_rules>
Every task MUST include:
1. <read_first> — Files executor MUST read
2. <acceptance_criteria> — Verifiable conditions
3. <action> — Concrete values, not references
</deep_work_rules>
```

**Plan Checker Verification Dimensions:**
```markdown
## Dimension 1: Requirement Coverage
## Dimension 2: Task Completeness
## Dimension 3: Dependency Correctness
## Dimension 4: Key Links Planned
## Dimension 5: Scope Sanity
## Dimension 6: Verification Derivation
## Dimension 7: Context Compliance
## Dimension 8: Nyquist Compliance
## Dimension 9: Cross-Plan Data Contracts
## Dimension 10: CLAUDE.md Compliance
```

**Gap Closure Mode:**
```markdown
**Triggered by `--gaps` flag.**
1. Find gap sources (VERIFICATION.md, UAT.md with diagnosed status)
2. Parse gaps (truth, reason, artifacts, missing)
3. Load existing SUMMARYs
4. Group gaps by artifact/concern/dependency
5. Create gap closure tasks
6. Assign waves
```

### Notable Implementation Details

**GOOD: Multiple Plan Modes**
Standard, gap_closure, reviews, revision - one planner agent handles all modes via mode-specific prompt injection.

**GOOD: PRD Express Path**
When `--prd path/to/prd.md` provided, skips discuss-phase entirely and generates CONTEXT.md from PRD. All PRD requirements become locked decisions.

**GOOD: Requirements Coverage Gate**
After plans pass checker, verifies ALL phase REQ-IDs appear in at least one plan's requirements field. Coverage must be 100%.

**GOOD: 10-Dimensional Verification**
Comprehensive plan checking before execution burns context:
- Dimensions 1-6: Core plan quality
- Dimension 7: Context compliance (locked decisions honored)
- Dimension 8: Nyquist (automated testing requirements)
- Dimension 9: Cross-plan data contracts
- Dimension 10: CLAUDE.md compliance

**GOOD: Wave 0 for Tests**
If task has `tdd="true"` and no test exists, action specifies:
```xml
<verify>
  <automated>MISSING — Wave 0 must create {test_file} first</automated>
</verify>
```

**GOOD: CLAUDE.md Compliance**
Plans must respect project-specific conventions from `./CLAUDE.md`. Checker verifies:
- Libraries/patterns forbidden in CLAUDE.md aren't used
- Required steps aren't skipped
- Code style matches conventions

**TECHNICAL DEBT: Windows Stdio Deadlock**
```markdown
**Windows users:** If plan-phase freezes during agent spawning (common on Windows
due to stdio deadlocks with MCP servers — see Claude Code issue #28126):
1. Force-kill terminal
2. Clean up orphaned node processes
3. Clean up stale task directories
4. Reduce MCP server count
5. Retry with --skip-research
```

**SHORTCUT: Auto-Advance Chain**
Uses Skill tool (not Task/Task) to avoid nested agent runtime freezes:
```markdown
Skill(skill="gsd:execute-phase", args="${PHASE} --auto --no-transition ${GSD_WS}")
```
The `--no-transition` flag keeps the chain flat rather than spawning deeper Task agents.

**SHORTCUT: Skip-Review Chain**
When `--auto` or config `workflow.auto_advance` is true, chain continues:
```
discuss → plan → execute → next discuss → ...
```
Without requiring manual intervention between phases.

---

## Cross-Cutting Technical Patterns

### Agent Spawning Pattern
All features use the same Task() spawning pattern:
```markdown
Task(
  prompt=filled_prompt,
  subagent_type="gsd-[role]",
  model="{model_from_init}",
  description="Description for logs"
)
```
Model resolution happens in init step via `gsd-tools`.

### Config-Driven Behavior
Workflow behavior heavily dependent on `.planning/config.json`:
- `workflow.research_before_questions`
- `workflow.discuss_mode` (discuss vs assumptions)
- `workflow.nyquist_validation`
- `workflow.ui_phase`, `workflow.ui_safety_gate`
- `workflow.auto_advance`
- `workflow.text_mode`

### File Path Accumulation Pattern
Features progressively accumulate file paths across steps:
- new-project: project_path → research files
- discuss-phase: phase_dir → context, discussion-log
- plan-phase: phase_dir → plans, verification, reviews

### Commit Pattern
All features commit immediately after creating files:
```bash
node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" commit \
  "docs(${phase}): [description]" --files "files..."
```
Atomic commits ensure artifacts persist even if context is lost.

---

## Technical Debt and Shortcuts Summary

| Feature | Issue | Severity | Type |
|--------|-------|----------|------|
| Project Init | No rollback if research partially fails | Medium | Error Handling |
| Project Init | Config set before greenfield/brownfield known | Low | Logic Order |
| Discuss | Cannot nest discuss-phase via Task() | High | Workflow Architecture |
| Discuss | Auto mode logs choices without real discussion | Low | UX Shortcut |
| Plan Phase | Windows MCP stdio deadlock | High | Platform Bug |
| Plan Phase | --auto chain uses --no-transition hack | Medium | Workaround |
| All | No distributed lock on .planning/ writes | Low | Concurrency |

---

## Flow Diagrams

### Project Initialization Flow
```
┌─────────────────────────────────────────────────────────────┐
│                    /gsd:new-project                        │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  1. Setup (init tool, git init if needed)                  │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  2. Brownfield? → Map codebase first or skip              │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  3. Deep Questioning OR Auto-config extraction            │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  4. Write PROJECT.md                                       │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  5. Workflow Preferences (mode, granularity, etc.)         │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  6. Research? → 4x gsd-project-researcher (parallel)      │
│                  → gsd-research-synthesizer               │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  7. Define Requirements (v1/v2/out-of-scope)             │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  8. Create Roadmap (gsd-roadmapper)                         │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  9. Done → COMMIT → Auto-advance to discuss-phase 1        │
└─────────────────────────────────────────────────────────────┘
```

### Discuss Phase Flow
```
┌─────────────────────────────────────────────────────────────┐
│                  /gsd:discuss-phase {N}                   │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  1. Init → Validate phase exists                           │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  2. Check existing CONTEXT.md? → Update/View/Skip          │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  3. Load Prior Context (PROJECT, REQUIREMENTS, prior phases)│
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  4. Cross-Reference TODOs → Fold into scope or defer        │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  5. Scout Codebase (or use existing maps)                   │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  6. Analyze Phase → Generate gray areas                    │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  7. Present Gray Areas → User selects which to discuss      │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  8. Advisor Research? (if ADVISOR_MODE)                   │
│     → Parallel research agents per area                     │
│     → Synthesize comparison tables                         │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  9. Discuss Areas → Q&A per area, scope creep guard        │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  10. Write CONTEXT.md + DISCUSSION-LOG.md                    │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  11. Commit + Update STATE.md                               │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  12. Auto-advance? → /gsd:plan-phase {N} --auto           │
└─────────────────────────────────────────────────────────────┘
```

### Plan Phase Flow
```
┌─────────────────────────────────────────────────────────────┐
│                   /gsd:plan-phase {N}                      │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  1. Init → Parse flags, validate phase                    │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  2. PRD Express? → Generate CONTEXT.md from PRD            │
│     (skip to step 5)                                        │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  3. Load CONTEXT.md → If missing, offer discuss-phase     │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  4. Research? → gsd-phase-researcher → VALIDATION.md       │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  5. UI Design Contract Gate (if phase has UI)              │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  6. Check Existing Plans → Add/View/Replan                 │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  7. Spawn gsd-planner → Create PLAN.md files               │
│     (goal-backward, 2-3 tasks, wave assignment)           │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  8. Handle Return → COMPLETE/CHECKPOINT/INCONCLUSIVE       │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  9. Verify? → gsd-plan-checker (10 dimensions)             │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  10. Issues? → Revision Loop (max 3)                       │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  11. Requirements Coverage Gate (100% REQ-ID coverage)      │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  12. Done → COMMIT → Auto-advance to /gsd:execute-phase   │
└─────────────────────────────────────────────────────────────┘
```

---

## End of Batch 1 Research
