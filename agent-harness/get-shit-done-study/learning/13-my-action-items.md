# My Action Items: Building a Similar Project

**Derived from:** 12-lessons-learned.md
**Purpose:** Prioritized next steps for applying GSD patterns to a new project

---

## Architecture

### [HIGH] Implement Context Budget Targeting
**Rationale:** The 50% context target is foundational to GSD's quality maintenance. Without it, long projects suffer quality degradation.

**Action:** Design context tracking into workflow orchestration from day one.
- Instrument agent spawning with context usage reporting
- Set thresholds: 50% target, 70% warning, 80% abort
- Log context metrics per task for retrospective analysis

**Reference:** GSD's `hooks/gsd-context-monitor.js` monitors context percentage and injects warnings below 35% (warning) and 25% (critical).

---

### [HIGH] Design Wave-Based Execution from Day One
**Rationale:** Parallel execution is key to speed. Retrofitting it is harder than designing it in.

**Action:** Plan dependency graphs before execution. Tools needed:
- Dependency parser for plan frontmatter (`depends_on` field)
- Wave calculator (max depth from roots)
- Parallel spawner for same-wave plans
- Lock mechanism for shared file access

**Reference:** `bin/lib/core.cjs` - `withPlanningLock()` for mutual exclusion, `--no-verify` for parallel git commits.

---

### [HIGH] Use Vertical Slice Organization
**Rationale:** Horizontal layering (all models, then all APIs, then all UIs) creates sequential bottlenecks. Vertical slices (feature end-to-end) enable parallelism.

**Action:**
- When planning phases, organize by feature not by layer
- Each plan should be independently deployable
- Shared code (utilities, types) in separate plans that run first

---

### [MEDIUM] Implement File-Based State Management
**Rationale:** Human-readable state files (`.planning/` directory) survive context resets and enable git-based versioning.

**Action:**
- Define state file schema: `STATE.md` (YAML frontmatter + body), `ROADMAP.md`, `PLAN.md`, `SUMMARY.md`
- Build tooling for state CRUD operations
- Use markdown for files that humans will read; JSON for machine-only state

**Reference:** `bin/lib/state.cjs` - `syncStateFrontmatter()` pattern for keeping YAML frontmatter in sync with body.

---

## Quality

### [HIGH] Build Comprehensive Test Suite First
**Rationale:** GSD's 43 test files with 70% coverage requirement catches regressions early. Tests also serve as executable documentation.

**Action:**
- Use Node.js built-in test runner (`node:test`) - zero external dependencies
- Create centralized test helpers (`createTempProject()`, `runTools()`, `cleanup()`)
- Mandate `beforeEach`/`afterEach` hooks for setup/cleanup (not `try/finally`)
- Enforce coverage threshold (70% is good starting point)

**Reference:** `tests/helpers.cjs`, `CONTRIBUTING.md` testing standards.

---

### [HIGH] Add TypeScript Instead of Plain JavaScript
**Rationale:** GSD's choice of JavaScript is defensible but not optimal. For a new project, TypeScript's static typing prevents subtle bugs in complex orchestration logic.

**Action:**
- Use TypeScript for core library modules
- Strict mode enabled
- JSDoc for public API documentation
- Runtime validation at boundaries (input from external sources)

**Note:** This is a departure from GSD's approach but appropriate for new projects.

---

### [MEDIUM] Implement Automated Formatting
**Rationale:** GSD's reliance on "no drive-by formatting" convention is fragile. Prettier + ESLint prevent style drift.

**Action:**
- Add ESLint with strict config
- Add Prettier with project conventions
- Pre-commit hook to validate formatting
- CI enforcement

**GSD's convention to honor:** Don't reformat code unrelated to your change.

---

### [MEDIUM] Address Empty Catch Blocks
**Rationale:** Silent error swallowing hides failures. Non-critical operations should either:
- Log warnings
- Return result objects with error fields
- Be explicitly documented as intentionally empty

**Action:**
- ESLint rule to flag empty catch blocks
- Exceptions only for truly non-critical operations (e.g., optional cleanup)
- Result object pattern for recoverable errors

**Reference:** GSD's `bin/lib/security.cjs` - `safeJsonParse()` returns `{ok, value?, error?}` rather than throwing.

---

## Security

### [HIGH] Implement Path Traversal Prevention
**Rationale:** Any system that reads/writes files based on user input is vulnerable. GSD's `validatePath()` pattern is sound.

**Action:**
```javascript
function validatePath(filePath, baseDir) {
  // 1. Null byte rejection
  if (filePath.includes('\0')) return false;
  // 2. Realpath resolution for symlinks
  const resolved = fs.realpathSync(path.resolve(baseDir));
  // 3. Containment check
  return normalizedPath.startsWith(resolved + path.sep);
}
```

**Reference:** `bin/lib/security.cjs` - `validatePath()` with `fs.realpathSync()` for symlink resolution.

---

### [HIGH] Add Prompt Injection Detection
**Rationale:** User-controlled text flowing into LLM prompts is an attack vector. Detection patterns needed for:
- Instruction override: "ignore previous instructions", "disregard your role"
- Role manipulation: "you are now a", "act as a"
- System prompt extraction: "print your system prompt"
- Hidden markers: `<system>`, `<assistant>`, `[SYSTEM]`

**Action:**
- Build allowlist-based scanner (blocklist is incomplete)
- Strip zero-width characters: `\u200B-\u200F`, `\u2028-\u202F`, `\uFEFF`
- Neutralize XML-like tags: `<system>` -> `＜system-text＞`
- Add CI scan for common injection patterns

**Reference:** `bin/lib/security.cjs` - `scanForInjection()` with 14 patterns.

---

### [MEDIUM] Validate External Input at Boundaries
**Rationale:** GSD's config key allowlisting is a good pattern. Any external input should be validated before use.

**Action:**
- Allowlist config keys rather than blocklist
- Validate phase numbers with strict regex
- Validate field names to prevent regex injection
- Size-limit JSON parsing to prevent DoS

**Reference:** `bin/lib/config.cjs` - `VALID_CONFIG_KEYS` Set, `validatePhaseNumber()`, `validateFieldName()`.

---

### [MEDIUM] Sanitize Git History Inputs
**Rationale:** Commit messages flow back into agent context on future reads. Unsanitized messages could be a stored prompt injection vector.

**Action:**
- Sanitize commit messages before git operations
- Scan for injection patterns
- Strip zero-width characters

**Reference:** `bin/lib/commands.cjs` - `cmdCommit()` calls `sanitizeForPrompt()` on messages.

---

## Workflow

### [HIGH] Implement Thin Orchestrator Pattern
**Rationale:** Orchestrators should load context and spawn agents - never do heavy lifting themselves. This keeps orchestrator context small.

**Action:**
- Workflows: context loading + agent spawning only
- Agents: specialized, spawned fresh per task
- No shared mutable state between orchestrator and agents
- Communication via files (not in-memory)

**Reference:** `workflows/plan-phase.md` orchestrates but delegates to specialized agents.

---

### [HIGH] Design for Fresh Context Per Agent
**Rationale:** GSD's 200k token fresh context per spawn eliminates context rot. Long-running orchestrators accumulate stale context.

**Action:**
- Agent tasks should be completable in a single spawn
- Large tasks should be split into multiple plans
- Context should be re-established from files, not carried in orchestrator memory
- Tool: History digest for selecting relevant prior context

**Reference:** `workflows/execute-phase.md` spawns fresh executors per plan.

---

### [MEDIUM] Add Checkpoint Patterns for Human Verification
**Rationale:** Not everything can be automated. Human checkpoints for:
- Visual/functional verification (checkpoint:human-verify)
- Implementation choices (checkpoint:decision)
- External service actions (checkpoint:human-action) - rare

**Action:**
- Design checkpoint types into workflow schema
- Block execution at checkpoints until human approval
- Provide clear verification instructions at each checkpoint

**Reference:** `workflows/verify-work.md` - UAT workflow with guided walkthroughs.

---

## Community (If Open Source)

### [MEDIUM] Create Issue and PR Templates
**Rationale:** GSD lacks templates, relying on implicit conventions. Templates improve signal-to-noise.

**Action:**
- Bug report template with reproduction steps
- Feature request template with use case
- PR template with test checklist
- Link to docs in every template

---

### [LOW] Document Maintainership
**Rationale:** GSD's single-owner model works but lacks clarity on merge rights and responsibilities.

**Action:**
- Create MAINTAINERS.md listing merge rights
- Create AUTHORS.md for contributor attributions
- Document release process
- Define security reporting path

---

## Implementation Priority Summary

| Priority | Category | Action Item |
|----------|----------|-------------|
| HIGH | Architecture | Context budget targeting |
| HIGH | Architecture | Wave-based execution |
| HIGH | Architecture | Vertical slice organization |
| HIGH | Quality | Comprehensive test suite first |
| HIGH | Quality | TypeScript (departing from GSD) |
| HIGH | Security | Path traversal prevention |
| HIGH | Security | Prompt injection detection |
| HIGH | Workflow | Thin orchestrator pattern |
| HIGH | Workflow | Fresh context per agent |
| MEDIUM | Architecture | File-based state management |
| MEDIUM | Quality | Automated formatting |
| MEDIUM | Quality | Address empty catch blocks |
| MEDIUM | Security | Input validation at boundaries |
| MEDIUM | Security | Sanitize git history inputs |
| MEDIUM | Workflow | Checkpoint patterns |
| MEDIUM | Community | Issue and PR templates |
| LOW | Community | Document maintainership |

---

## Key Files to Study Before Implementation

1. `bin/lib/core.cjs` - Error handling, path helpers, planning lock
2. `bin/lib/security.cjs` - Validation, sanitization, injection detection
3. `bin/lib/state.cjs` - State file management
4. `workflows/execute-phase.md` - Wave orchestration
5. `workflows/plan-phase.md` - Planning orchestration
6. `agents/gsd-executor.md` - Executor agent definition
7. `tests/helpers.cjs` - Test utilities pattern
8. `CONTRIBUTING.md` - Project conventions
