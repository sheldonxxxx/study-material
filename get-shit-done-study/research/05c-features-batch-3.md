# Feature Batch 3 Analysis

## Features Covered
- Feature 7: Codebase Mapping
- Feature 8: Multi-Agent Orchestration
- Feature 9: Context Engineering System

---

## Feature 7: Codebase Mapping

### Description
Analyzes existing codebase before new-project to understand stack, architecture, conventions, and concerns. Spawns parallel agents for each analysis area.

### Core Implementation Files
- `get-shit-done/workflows/map-codebase.md` (workflow orchestrator)
- `agents/gsd-codebase-mapper.md` (agent definition)
- `get-shit-done/templates/codebase/*.md` (document templates)
- `get-shit-done/bin/lib/init.cjs` (cmdInitMapCodebase function, lines 764-796)

### Flow Trace

```
/gsd:map-codebase invoked
    ↓
map-codebase workflow (init.cjs: cmdInitMapCodebase)
    ↓
Check if .planning/codebase/ exists
    ↓
Create .planning/codebase/ directory
    ↓
Spawn 4 parallel gsd-codebase-mapper agents (if Task tool available)
    ├── Agent 1: Tech Focus → STACK.md, INTEGRATIONS.md
    ├── Agent 2: Architecture Focus → ARCHITECTURE.md, STRUCTURE.md
    ├── Agent 3: Quality Focus → CONVENTIONS.md, TESTING.md
    └── Agent 4: Concerns Focus → CONCERNS.md
    ↓
Wait for agents via TaskOutput
    ↓
Verify all 7 documents created
    ↓
Secret scan before commit
    ↓
git commit the documents
```

### Key Implementation Details

**Parallel Agent Spawning (map-codebase.md, lines 101-192):**
```markdown
Task(
  subagent_type="gsd-codebase-mapper",
  model="{mapper_model}",
  run_in_background=true,
  prompt="Focus: tech

Analyze this codebase for technology stack...

Write these documents to .planning/codebase/:
- STACK.md
- INTEGRATIONS.md
..."
```

**Runtime Capability Detection (map-codebase.md, lines 91-99):**
```markdown
<step name="detect_runtime_capabilities">
Before spawning agents, detect whether the current runtime supports the `Task` tool...

**CRITICAL:** Never use `browser_subagent` or `Explore` as a substitute for `Task`.
If `Task` is unavailable, perform the mapping sequentially in-context.
```

**Sequential Fallback (map-codebase.md, lines 228-257):**
When Task tool is unavailable (e.g., Antigravity, Gemini CLI, Codex), the workflow performs all 4 mapping passes sequentially in the current context instead of spawning subagents.

**Forbidden Files Pattern (gsd-codebase-mapper.md, lines 725-745):**
```markdown
<forbidden_files>
**NEVER read or quote contents from these files (even if they exist):**
- `.env`, `.env.*`, `*.env`
- `credentials.*`, `secrets.*`, `*secret*`
- `*.pem`, `*.key`, `*.p12`
- SSH private keys, NPMRC tokens, etc.
</forbidden_files>
```

### Notable Implementation Details

**1. Clever: Document Quality Over Brevity**
The agent instructions explicitly state: "A 200-line TESTING.md with real patterns is more valuable than a 74-line summary" (gsd-codebase-mapper.md, line 64).

**2. Clever: Sequential Fallback Pattern**
The workflow gracefully degrades to sequential in-context mapping when the Task tool is unavailable, ensuring compatibility across different AI runtimes.

**3. Security: Secret Scanning Before Commit**
Before committing the generated documents, the workflow runs a secret pattern scan:
```bash
grep -E '(sk-[a-zA-Z0-9]{20,}|sk_live_|ghp_|gho_|glpat-|AKIA|xox[baprs]-|
-----BEGIN.*PRIVATE KEY|eyJ[a-zA-Z0-9_-]+\.eyJ[a-zA-Z0-9_-]+\.)' \
  .planning/codebase/*.md
```

**4. Technical Debt: None Observed**
The codebase mapping feature appears well-designed with proper separation of concerns.

---

## Feature 8: Multi-Agent Orchestration

### Description
Thin orchestrator pattern that spawns specialized agents (researchers, planners, executors, verifiers) and routes results between stages. Keeps main context at 30-40%.

### Core Implementation Files
- `agents/gsd-planner.md` (planner agent - 1355 lines)
- `agents/gsd-executor.md` (executor agent - 510 lines)
- `agents/gsd-codebase-mapper.md` (mapper agent - 771 lines)
- `get-shit-done/references/model-profiles.md` (model tiering)
- `get-shit-done/bin/lib/init.cjs` (model resolution via resolveModelInternal)

### Agent Registry
The system defines 17 specialized agents in `agents/`:
- gsd-planner, gsd-executor, gsd-verifier, gsd-debugger
- gsd-project-researcher, gsd-phase-researcher, gsd-research-synthesizer
- gsd-codebase-mapper, gsd-roadmapper
- gsd-plan-checker, gsd-integration-checker, gsd-nyquist-auditor
- gsd-assumptions-analyzer, gsd-advisor-researcher
- gsd-ui-auditor, gsd-ui-researcher, gsd-ui-checker
- gsd-user-profiler

### Flow Trace

**Typical Planning Flow (plan-phase.md):**
```
/gsd:plan-phase invoked
    ↓
plan-phase workflow: init → load context → (optional research)
    ↓
Spawn gsd-phase-researcher (if research enabled)
    ↓
Researcher writes RESEARCH.md
    ↓
Spawn gsd-planner
    ↓
Planner writes PLAN.md files
    ↓
Spawn gsd-plan-checker (verification)
    ↓
(If issues: revision loop, max 3 iterations)
    ↓
Plans ready for execution
```

**Model Resolution (init.cjs, resolveModelInternal):**
```javascript
// From init.cjs - resolve model for agent
researcher_model: resolveModelInternal(cwd, 'gsd-phase-researcher'),
planner_model: resolveModelInternal(cwd, 'gsd-planner'),
checker_model: resolveModelInternal(cwd, 'gsd-plan-checker'),
```

### Model Profiles (model-profiles.md)

| Agent | `quality` | `balanced` | `budget` | `inherit` |
|-------|-----------|------------|----------|-----------|
| gsd-planner | opus | opus | sonnet | inherit |
| gsd-executor | opus | sonnet | sonnet | inherit |
| gsd-phase-researcher | opus | sonnet | haiku | inherit |
| gsd-codebase-mapper | sonnet | haiku | haiku | inherit |
| gsd-verifier | sonnet | sonnet | haiku | inherit |

**Design Philosophy:**
- `quality`: Maximum reasoning - Opus for all decision-making, Sonnet for read-only verification
- `balanced` (default): Smart allocation - Opus only for planning, Sonnet for execution/research
- `budget`: Minimal Opus - Sonnet for code writers, Haiku for research/verification
- `inherit`: Follow the current session model (required for non-Anthropic providers)

### Notable Implementation Details

**1. Clever: Thin Orchestrator Pattern**
The orchestrator never does heavy lifting. It coordinates by:
- Spawning specialized agents
- Routing results between stages
- Validating outputs
- Never implementing features directly

**2. Clever: Model Tiering by Task Complexity**
Different agents get different model tiers based on their cognitive demands:
- Planning (architecture decisions) → Opus
- Execution (follows explicit instructions) → Sonnet
- Research/Mapping (read-only exploration) → Haiku

**3. Clever: Agent Skills System (init.cjs, lines 1363-1420)**
```javascript
function buildAgentSkillsBlock(config, agentType, projectRoot) {
  // Reads config.agent_skills[agentType]
  // Validates skill paths exist
  // Returns formatted skills block for injection into Task prompts
}
```
This allows users to inject project-specific skills into any agent type.

**4. Technical Debt: Potential Context Contamination**
The model-profiles note (line 136): "GSD returns `"inherit"` for opus-tier agents, causing them to use whatever opus version the user has configured." This is a workaround for version conflicts but creates non-deterministic behavior.

**5. Design: Non-Blocking Orchestrator**
Agents spawn with `run_in_background=true` and the orchestrator uses `TaskOutput` to collect results, allowing true parallelism.

**6. Error Handling: Graceful Degradation**
When agent spawn fails or agents produce incomplete output, the workflow notes failures and continues with successful results (map-codebase.md, lines 223-224).

---

## Feature 9: Context Engineering System

### Description
File-based context management using PROJECT.md, research/, REQUIREMENTS.md, ROADMAP.md, STATE.md, PLAN.md, SUMMARY.md to prevent context rot across sessions.

### Core Implementation Files
- `get-shit-done/references/planning-config.md` (planning directory behavior)
- `get-shit-done/references/continuation-format.md` (next steps formatting)
- `get-shit-done/templates/summary.md` (phase summary template)
- `get-shit-done/templates/state.md` (state template)
- `get-shit-done/templates/context.md` (context template)
- `get-shit-done/templates/project.md` (project template)
- `get-shit-done/templates/roadmap.md` (roadmap template)
- `get-shit-done/bin/lib/init.cjs` (file existence checks, paths)

### Context File System Structure

```
.planning/
├── PROJECT.md          # Project definition, value, constraints
├── ROADMAP.md         # Phase definitions with goals
├── STATE.md           # Current position, metrics, accumulated context
├── REQUIREMENTS.md    # Requirement traceability
├── config.json        # Planning behavior configuration
├── MILESTONES.md      # Milestone tracking
├── codebase/          # Codebase analysis maps
│   ├── STACK.md
│   ├── ARCHITECTURE.md
│   ├── STRUCTURE.md
│   ├── CONVENTIONS.md
│   ├── TESTING.md
│   ├── INTEGRATIONS.md
│   └── CONCERNS.md
├── phases/
│   └── XX-name/
│       ├── XX-CONTEXT.md       # User decisions from discuss-phase
│       ├── XX-RESEARCH.md      # Technical research
│       ├── XX-VALIDATION.md    # Verification strategy
│       ├── XX-UI-SPEC.md       # UI design contract
│       ├── XX-PLAN.md          # Implementation plan
│       ├── XX-SUMMARY.md       # Execution results
│       ├── XX-VERIFICATION.md # Code verification
│       └── XX-UAT.md           # User acceptance testing
├── todos/
│   ├── pending/
│   └── completed/
└── archive/
```

### Planning Config Schema (planning-config.md)

```json
{
  "planning": {
    "commit_docs": true,        // Whether to commit planning files to git
    "search_gitignored": false  // Add --no-ignore to broad searches
  },
  "git": {
    "branching_strategy": "none",        // "none", "phase", or "milestone"
    "phase_branch_template": "gsd/phase-{phase}-{slug}",
    "milestone_branch_template": "gsd/{milestone}-{slug}",
    "quick_branch_template": null
  }
}
```

### Context Flow in Planning

**State Management Flow:**
1. **Create**: ROADMAP.md created, STATE.md initialized from template
2. **Read**: Every workflow reads STATE.md first for position/context
3. **Update**: After execution, STATE.md updated with:
   - Position (phase, plan, status)
   - Decisions (extracted from summaries)
   - Blockers/concerns
   - Performance metrics

**Summary Dependency Graph (summary.md frontmatter):**
```yaml
requires:
  - phase: [prior phase]
    provides: [what that phase built]
provides:
  - [bullet list of deliverables]
affects: [phases that will need this context]
```

This enables transitive context closure during planning - when planning a new phase, the system can trace back through all dependencies.

### Continuation Format (continuation-format.md)

Standard format for presenting next steps:
```markdown
---

## ▶ Next Up

**{identifier}: {name}** — {one-line description}

`{command to copy-paste}`

<sub>`/clear` first → fresh context window</sub>

---

**Also available:**
- {alternative option 1}
- {alternative option 2}

---
```

### Key Implementation Details

**1. Clever: File-Based Context Prevents Loss**
All context lives in files that persist across sessions. If Claude Code crashes or user switches sessions, work is preserved in `.planning/`.

**2. Clever: Context Budget Enforcement**
The system enforces ~50% context target per plan. Each plan:
- Has 2-3 tasks maximum
- Should complete within ~50% of token budget
- Creates a fresh SUMMARY for downstream consumption

**3. Clever: Dependency Graph Enables Context Selection**
When planning, the system can automatically load relevant summaries by tracing `requires/provides/affects` in frontmatter.

**4. Planning: Auto-Detection of Gitignored .planning/**
From planning-config.md (line 58):
```markdown
**Auto-detection:** If `.planning/` is gitignored, `commit_docs` is automatically `false`
regardless of config.json. This prevents git errors when users have `.planning/` in `.gitignore`.
```

**5. Design: Separation of Concerns**
- `PROJECT.md`: What we're building, why, constraints
- `ROADMAP.md`: Phases and their goals
- `STATE.md`: Current position and short-term memory
- `CONTEXT.md`: User decisions for a specific phase
- `RESEARCH.md`: Technical investigation results
- `PLAN.md`: Implementation tasks
- `SUMMARY.md`: Execution results and learnings

**6. Technical Debt: Potential Staleness**
The context files can become stale if:
- User manually edits code without updating SUMMARY
- Multiple sessions run concurrently with different views
- Plan-phase doesn't fully consume prior summaries

**7. Design: Size Constraint on STATE.md**
The state template explicitly limits STATE.md to 100 lines:
```markdown
<size_constraint>
Keep STATE.md under 100 lines.

It's a DIGEST, not an archive. If accumulated context grows too large:
- Keep only 3-5 recent decisions in summary
- Keep only active blockers, remove resolved ones
</size_constraint>
```

---

## Cross-Feature Observations

### Integration Points

1. **Codebase Mapping → Planning**: When `/gsd:plan-phase` runs, it loads relevant codebase documents based on phase type:
   - UI phases load CONVENTIONS.md, STRUCTURE.md
   - API phases load ARCHITECTURE.md, CONVENTIONS.md
   - Database phases load ARCHITECTURE.md, STACK.md

2. **Multi-Agent → Context**: Agents write directly to context files:
   - Researchers write RESEARCH.md
   - Planners write PLAN.md
   - Executors write SUMMARY.md

3. **Context → Execution**: The executor reads:
   - STATE.md for current position
   - PLAN.md for tasks
   - Prior SUMMARYs for context

### Shared Patterns

**1. File-Based State**: All three features use files as the source of truth rather than in-memory state.

**2. Structured Frontmatter**: PLAN.md and SUMMARY.md use YAML frontmatter for machine-readable metadata.

**3. Context Routing**: The init.cjs module centralizes context assembly, providing file paths and existence flags to workflows.

**4. Secret Scanning**: Codebase mapping includes secret scanning before commit, a security pattern that could be applied more broadly.

---

## Summary

| Aspect | Feature 7 | Feature 8 | Feature 9 |
|--------|-----------|-----------|-----------|
| **Purpose** | Analyze existing code | Coordinate specialized agents | Prevent context loss |
| **Key Innovation** | Parallel mapping with fallback | Model tiering by task | File-based dependency graph |
| **Strength** | Runtime compatibility | Cognitive resource optimization | Session resilience |
| **Weakness** | Large codebase = many docs | Agent proliferation risk | Potential staleness |
| **Lines of Code** | ~378 (workflow + agent) | ~2636 (all agents) | ~500 (templates + config) |
