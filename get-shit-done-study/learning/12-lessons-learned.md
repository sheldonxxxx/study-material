# Lessons Learned: get-shit-done Study

**Study Date:** 2026-03-26
**Research Source:** 8 research documents analyzing topology, tech stack, community, features, architecture, code quality, and security

---

## What to Emulate

### 1. Context Budget Management
**Pattern:** Target 50% context usage per plan, fresh 200k tokens per executor spawn.

**Why it works:** Eliminates context rot in long projects. The "Quality Degradation Curve" (0-30% PEAK, 30-50% GOOD, 50-70% DEGRADING, 70%+ POOR) provides a concrete measurement framework.

**Implementation reference:** `get-shit-done/workflows/execute-phase.md` spawns fresh executors per plan rather than reusing context.

---

### 2. Wave-Based Parallel Execution
**Pattern:** Analyze plan dependencies, group into waves, execute waves sequentially but plans within waves in parallel.

**Why it works:** Maximizes parallelism while respecting dependencies. Parallel commits use `--no-verify` to avoid hook contention; STATE.md uses file locking for mutual exclusion.

**Implementation reference:** `get-shit-done/bin/lib/core.cjs` - `withPlanningLock()` provides atomic lock acquisition with stale lock recovery (30s timeout).

---

### 3. Vertical Slice Organization
**Pattern:** Plans organized by feature end-to-end (model + API + UI), not horizontal layers (all models, all APIs, all UIs).

**Why it works:** Each slice is self-contained and can execute independently. Horizontal layering creates sequential bottlenecks.

**Example:** Plan 01 = User feature (model + API + UI), Plan 02 = Product feature (model + API + UI). Both run in Wave 1.

---

### 4. Atomic Commits Per Task
**Pattern:** Each task gets its own commit for bisect-friendly history.

**Why it works:** Enables surgical rollback to specific task failures without reverting unrelated changes. Combined with wave-based execution, provides fine-grained progress tracking.

**Implementation reference:** `get-shit-done/bin/lib/core.cjs` - `cmdCommit()` with commit message sanitization for prompt injection prevention.

---

### 5. Security-First Input Validation
**Pattern:** Comprehensive runtime validation: path traversal prevention, prompt injection detection, shell argument validation, JSON size limits.

**Why it works:** GSD's threat model (user-controlled text flowing into LLM prompts) is real. 14 prompt injection patterns detected, path traversal uses `fs.realpathSync()` for symlink resolution.

**Implementation reference:** `get-shit-done/bin/lib/security.cjs`:
```javascript
function validatePath(filePath, baseDir, opts = {}) {
  // Null byte rejection
  if (filePath.includes('\0')) { return { safe: false, ... }; }
  // Realpath resolution to resolve symlinks
  resolvedBase = fs.realpathSync(path.resolve(baseDir));
  // Containment check
  if (!normalizedPath.startsWith(normalizedBase)) { return { safe: false, ... }; }
}
```

---

### 6. Thin Orchestrator Pattern
**Pattern:** Workflows load context and spawn agents; they never do heavy lifting themselves.

**Why it works:** Keeps orchestrator context at 30-40%, preventing quality degradation. Heavy lifting delegated to specialized agents with fresh contexts.

**Implementation reference:** `get-shit-done/workflows/plan-phase.md` - orchestrates but delegates to gsd-planner, gsd-phase-researcher, gsd-plan-checker agents.

---

### 7. File-Based State Management
**Pattern:** All state in `.planning/` as human-readable Markdown/JSON files (STATE.md, ROADMAP.md, PLAN.md, SUMMARY.md).

**Why it works:** Survives context resets, inspectable by humans and agents, enables git-based versioning of planning artifacts.

**Implementation reference:** `get-shit-done/bin/lib/state.cjs` - `syncStateFrontmatter()` synchronizes YAML frontmatter with body text.

---

### 8. Comprehensive Test Suite with Minimal Dependencies
**Pattern:** Node.js built-in test runner (`node:test`), 43 test files, 70% coverage target, centralized helpers.

**Why it works:** Zero external test framework dependencies. Tests must pass on Node 22 (LTS) and Node 24 (Current).

**Implementation reference:** `tests/helpers.cjs` provides `createTempProject()`, `runGsdTools()`, `cleanup()` utilities. CONTRIBUTING.md mandates `beforeEach`/`afterEach` hooks over `try/finally` for cleanup.

---

### 9. Feature Flags with "Absent = Enabled"
**Pattern:** `workflow.nyquist_validation: false` in config disables; absent key defaults to enabled.

**Why it works:** Reduces configuration burden. Users opt-out rather than opt-in.

**Implementation reference:** `get-shit-done/bin/lib/config.cjs` - `loadConfig()` applies defaults when keys are absent.

---

### 10. Prompt Injection Defense-in-Depth
**Pattern:** Claude Code hooks (gsd-prompt-guard.js) scan writes to `.planning/` files, sanitization functions neutralize injection patterns.

**Why it works:** User text (PRDs, commit messages) flows into LLM prompts. Zero-width characters stripped, XML-like tags neutralized (`<system>` -> `＜system-text＞`), bracket markers neutralized (`[SYSTEM]` -> `[SYSTEM-TEXT]`).

**Implementation reference:** `hooks/gsd-prompt-guard.js` - advisory warning (not blocking) on detected injection patterns.

---

## What to Avoid

### 1. No TypeScript for Complex Architecture
**Problem:** Despite 19 core library modules and sophisticated multi-agent orchestration, the project uses plain JavaScript.

**Impact:** Runtime validation compensates but cannot catch type mismatches at build time. Function signatures not enforceable.

**Mitigation applied:** Extensive runtime validation in `security.cjs`, but this is defensive rather than preventive.

---

### 2. No Automated Linting/Formatting
**Problem:** No ESLint, no Prettier, no `.eslintrc*` anywhere in the repository.

**Impact:** Code style relies on developer discipline. CONTRIBUTING.md says "no drive-by formatting" but cannot enforce it.

**Risk:** Inconsistent formatting across contributors despite conventional commits requirement.

---

### 3. Empty Catch Blocks Hiding Errors
**Problem:** Common pattern in codebase:
```javascript
try {
  fs.writeFileSync(configPath, JSON.stringify(parsed, null, 2), 'utf-8');
} catch { /* intentionally empty */ }
```

**Impact:** Non-critical failures silently ignored. Can mask configuration problems.

**CONTRIBUTING.md:** Does not explicitly address catch block style.

---

### 4. Advisory-Only Prompt Guard
**Problem:** `gsd-prompt-guard.js` does NOT block suspicious content - only warns.

**Risk:** Sophisticated prompt injection could pass through or rely on agent not noticing warning.

**Documentation acknowledges:** "defense-in-depth, not a security boundary."

---

### 5. Config Initialization Order Issue
**Problem:** Config created BEFORE knowing if project is greenfield or brownfield.

**From research:** "If user declines research later, model_profile set in config may not match what they would have chosen interactively."

**Impact:** Potential mismatch between user intent and stored configuration.

---

### 6. No Rollback on Research Failure
**Problem:** If 4 parallel researchers spawn but one fails, the synthesizer still runs with partial results.

**Impact:** Partial/inconsistent research feeds into roadmap creation. No explicit error handling for partial research completion.

---

### 7. Single Owner Model
**Problem:** CODEOWNERS gives only @glittercowboy review authority.

**Impact:** Bottleneck potential. No maintainer documentation for who else has merge rights.

**Acknowledged gap:** Community report notes "no MAINTAINERS file, no AUTHORS file."

---

## Surprises

### 1. Pure CommonJS with Zero Runtime Dependencies
**Finding:** Despite sophisticated multi-agent architecture, the project uses:
- No TypeScript
- No ESM
- No runtime dependencies (only 2 dev dependencies: c8, esbuild)
- All `.cjs` files with `require()` syntax

**Implication:** Complexity managed through architecture, not tooling. The project's sophistication is in its design patterns, not its dependency tree.

---

### 2. Massive Community Engagement
**Finding:**
- 42,324 stars, 3,422 forks
- 488 open issues, 69 open PRs
- 488 commits from unique contributors in 30 days
- International documentation (Korean, Japanese, Portuguese, Chinese)
- 2-3 releases per week

**Implication:** The "meta-prompting/context engineering" niche has genuine market fit. Solo developer (with community) can sustain active development.

---

### 3. Comprehensive Security Despite Local CLI Nature
**Finding:** Security measures include:
- Path traversal prevention with symlink resolution
- 14-pattern prompt injection detection
- Shell argument validation
- JSON size limits (1MB default)
- Config key allowlisting
- Commit message sanitization
- CI security scanning (secret scan, prompt injection scan, base64 scan)

**Implication:** Security is not proportional to "exposure" - it's proportional to "what could go wrong." User-controlled text flowing into LLM prompts is a legitimate attack vector.

---

### 4. Quality Degradation Curve as First-Class Concept
**Finding:** The workflow documentation explicitly defines context usage thresholds:
- 0-30%: PEAK quality
- 30-50%: GOOD quality
- 50-70%: DEGRADING quality
- 70%+: POOR quality

**Implication:** Treating context as a scarce resource with measurable impact. Plans target ~50% context to leave room for complexity.

---

### 5. Markdown Normalization as Core Feature
**Finding:** `normalizeMd()` in `core.cjs` automatically formats generated `.planning/` files:
- MD022: Blank lines around headings
- MD031: Blank lines around fenced code blocks
- MD032: Blank lines around lists
- MD012: No 3+ consecutive blank lines
- MD047: Files end with single newline

**Implication:** Generated code is always IDE-friendly. No merge conflicts from formatting differences.

---

### 6. Cross-Platform as Baseline
**Finding:** CI matrix tests on:
- Ubuntu (latest), macOS (latest), Windows (latest)
- Node 22 (LTS) and Node 24 (Current)
- Supports Claude Code, OpenCode, Gemini CLI, Codex, Copilot, Antigravity

**Implication:** Platform fragmentation is a real cost. GSD's installer-mediated transformation handles multiple runtimes, but testing burden is significant.

---

## Honest Assessment

### Strengths
1. **Cohesive architecture** - Multi-agent orchestration with thin coordinators, fresh contexts, wave-based execution
2. **Security-conscious** - Defense-in-depth for prompt injection, path traversal, shell injection
3. **Developer experience** - Comprehensive CONTRIBUTING.md, centralized test helpers, conventional commits
4. **Community traction** - 42k stars validates the niche
5. **Minimal dependencies** - Zero runtime deps proves design sophistication over tooling reliance

### Weaknesses
1. **No static typing** - Type safety traded for simplicity
2. **No automated formatting** - Style relies on discipline
3. **Single point of failure** - Owner model creates review bottleneck
4. **Partial error handling** - Empty catches, no research rollback
5. **Advisory security** - Prompt guard warns but doesn't block

### Verdict
**get-shit-done** is a well-architected, actively maintained project that solves a real problem (context management for AI-assisted development). Its trade-offs (JavaScript over TypeScript, runtime validation over static typing) are intentional and defensible for a CLI tool. The security measures are more comprehensive than most projects of similar scope, reflecting genuine understanding of the LLM integration threat model.

The project demonstrates that **architecture sophistication can compensate for tooling simplicity** - but only when backed by rigorous testing (70% coverage), security scanning (CI), and community engagement (frequent releases, international docs).

---

## Specific Code References

| Pattern | File | Lines |
|---------|------|-------|
| Context budget target | `workflows/execute-phase.md` | N/A (in documentation) |
| Planning lock | `bin/lib/core.cjs` | ~200-225 |
| Path validation | `bin/lib/security.cjs` | `validatePath()` |
| Prompt injection scan | `bin/lib/security.cjs` | `scanForInjection()` |
| Sanitization | `bin/lib/security.cjs` | `sanitizeForPrompt()` |
| Shell arg validation | `bin/lib/security.cjs` | `validateShellArg()` |
| Config defaults | `bin/lib/config.cjs` | `buildNewProjectConfig()` |
| State frontmatter sync | `bin/lib/state.cjs` | `syncStateFrontmatter()` |
| Markdown normalization | `bin/lib/core.cjs` | `normalizeMd()` |
| Test helpers | `tests/helpers.cjs` | `createTempProject()` |
| Prompt guard hook | `hooks/gsd-prompt-guard.js` | PreToolUse handler |
