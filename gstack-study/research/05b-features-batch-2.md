# Features Deep Dive: Batch 2

**Repository:** `/Users/sheldon/Documents/claw/reference/gstack`
**Features:** Review (PR Review), QA (Quality Assurance), Ship (Release Engineer)
**Analyzed:** 2026-03-27

---

## Feature 1: Review (PR Review)

### Purpose
Staff engineer PR review. Analyzes diff against the base branch for SQL safety, LLM trust boundary violations, conditional side effects, and other structural issues that pass CI but blow up in production. Auto-fixes obvious issues, flags completeness gaps.

### Core Implementation Files
- `review/SKILL.md` (1,062 lines) — Main skill definition
- `review/checklist.md` (204 lines) — The review checklist with 11 categories
- `review/greptile-triage.md` (221 lines) — Greptile integration for external review comments
- `review/design-checklist.md` (133 lines) — Lite design review for frontend changes
- `review/TODOS-format.md` (63 lines) — Canonical TODOS.md format
- `bin/gstack-review-log` — Persists review results to `~/.gstack/projects/{slug}/{branch}-reviews.jsonl`
- `bin/gstack-review-read` — Reads review log for dashboard display

### Workflow Flow

```
Step 0: Detect platform (GitHub/GitLab/unknown) + base branch
    ↓
Step 1: Check branch — abort if on base or no diff
    ↓
Step 1.5: Scope Drift Detection
    ├─ Read TODOS.md, PR description, commit messages
    ├─ Discover plan file (conversation context → grep fallback → recent 24h)
    ├─ Extract actionable items from plan (max 50)
    └─ Cross-reference against diff: DONE/PARTIAL/NOT DONE/CHANGED
    ↓
Step 2: Read checklist.md
    ↓
Step 2.5: Greptile comment fetch (if PR exists)
    ├─ Fetch line-level + top-level comments via GitHub API
    ├─ Check suppressions (per-project + global greptile-history.md)
    └─ Classify: VALID & ACTIONABLE / VALID BUT ALREADY FIXED / FALSE POSITIVE / SUPPRESSED
    ↓
Step 3: Get diff (fetch base branch first)
    ↓
Step 4: Two-pass review
    ├─ Pass 1 (CRITICAL): SQL & Data Safety, Race Conditions, LLM Trust Boundary, Enum & Value Completeness
    └─ Pass 2 (INFORMATIONAL): 7 more categories
    ↓
Step 4.5: Design Review (conditional, only if frontend files changed)
    ├─ Read DESIGN.md if exists
    ├─ Apply design-checklist.md (6 AI slop + 4 typography + 4 spacing + 3 interaction + 3 violations)
    └─ Optional Codex design voice
    ↓
Step 4.75: Test Coverage Diagram
    ├─ Detect test framework
    ├─ Trace every codepath in diff
    ├─ Map user flows + interaction edge cases
    ├─ Check existing tests (quality scoring: ★ to ★★★)
    └─ Generate ASCII coverage diagram
    ↓
Step 5: Fix-First Review
    ├─ Classify each finding: AUTO-FIX vs ASK
    ├─ Auto-fix all AUTO-FIX items
    ├─ Batch-ask about ASK items
    └─ Apply user-approved fixes
    ↓
Step 5.5: TODOS.md cross-reference
Step 5.6: Documentation staleness check
Step 5.7: Adversarial review (auto-scaled by diff size)
    ├─ Small (<50 lines): Skip
    ├─ Medium (50-199): Codex adversarial OR Claude subagent
    └─ Large (200+): All 3 passes
    ↓
Step 5.8: Persist review result to log
```

### Key Design Patterns

**Fix-First Heuristic:** Mechanical fixes (dead code, N+1, magic numbers) are auto-fixed without asking. Complex issues (security, race conditions, design decisions) require user judgment.

**Plan File Discovery:** Three-tier fallback for finding the relevant plan:
1. Conversation context (plan file path in system messages)
2. Content-based grep (branch name → repo name → recent 24h)
3. Validation pass before using

**Scope Drift Detection:** Compares what was planned vs what landed. NOT DONE items become MISSING REQUIREMENTS evidence. Items not in plan become SCOPE CREEP evidence.

**Adversarial Review Auto-Scaling:**
- Diff size determines thoroughness (small/medium/large)
- Cross-model synthesis when multiple passes run
- Gaps in coverage get flagged for follow-up

**Greptile Integration:**
- Fetches comments from GitHub API (line-level + top-level in parallel)
- Two-tier reply system (friendly first, firm for escalation)
- Per-project and global suppression history
- Categorizes: race-condition, null-check, error-handling, style, type-safety, security, performance, correctness

### Notable Clever Solutions

1. **Enum & Value Completeness reads outside the diff** — The only category that requires tracing new values through all consumers. Uses grep to find sibling values, then reads each consumer file to verify completeness.

2. **Greptile suppression by category** — Suppressions match on repo + file pattern + category, allowing per-issue-type suppression without suppressing all comments on a file.

3. **Review Readiness Dashboard** — Aggregates runs/status/findings across all review skills (CEO, Eng, Design, Adversarial, Codex) with staleness detection.

4. **WTF-likelihood in QA (reused pattern)** — Not in /review but the same self-regulation pattern exists in /qa. Tracks revert rate to prevent endless fix loops.

### Edge Cases / Error Handling

- No PR exists: Greptile step skipped silently
- Checklist unreadable: STOP (blocks review)
- Plan file found but unreadable: skip with note
- Codex auth failure: falls back to Claude subagent
- Greptile API error: skip silently
- All suppressions stored as history, never deleted

### Technical Debt / Shortcuts

1. **Fix-First without revert path** — Auto-fixes are applied directly. If an auto-fix introduces a regression, there's no automatic revert mechanism. The reviewer must catch it manually.

2. **Greptile position=null filter** — Handles force-pushes but comments on deleted lines are silently dropped (acceptable tradeoff).

3. **Adversarial subagent prompt injection risk** — The Agent tool prompt includes "Run `git diff origin/<base>`" which could behave unexpectedly if git state is complex.

4. **Coverage diagram is ASCII text** — No programmatic parsing. The human reads it to understand coverage. Gaps are listed but not tracked systematically.

---

## Feature 2: QA (Quality Assurance)

### Purpose
QA Lead that tests the app, finds bugs, fixes them with atomic commits, and auto-generates regression tests for every fix. Three tiers: Quick (critical/high only), Standard (+ medium), Exhaustive (+ cosmetic). Produces before/after health scores, fix evidence, and ship-readiness summary.

### Core Implementation Files
- `qa/SKILL.md` (1,058 lines) — Main skill definition
- `qa/references/issue-taxonomy.md` (86 lines) — Severity levels + categories
- `qa/templates/qa-report-template.md` (127 lines) — Report format template
- Browse tool integration (`$B` command)

### Workflow Flow

```
Phase 1: Initialize
    ├─ Find browse binary (local or ~/.claude/skills/gstack)
    ├─ Detect platform + base branch
    ├─ Bootstrap test framework if needed
    └─ Create output directories
    ↓
Phase 2: Authenticate (if needed)
    ├─ Cookie import OR
    ├─ Form fill + snapshot verify OR
    └─ 2FA/OTP wait
    ↓
Phase 3: Orient
    ├─ Snapshot + annotated screenshot
    ├─ Map links / navigation structure
    └─ Console errors check
    ↓
Phase 4: Explore
    ├─ Systematic page visit
    ├─ Per-page checklist: visual → interactive → forms → nav → states → console → responsiveness
    └─ Depth judgment (core features get more time)
    ↓
Phase 5: Document
    └─ Issue-by-issue evidence capture (immediate, not batched)
    ↓
Phase 6: Wrap Up
    ├─ Compute health score
    ├─ Write baseline.json
    └─ Regression mode diff if baseline exists
    ↓
Phase 7: Triage
    └─ Sort by severity, filter by tier
    ↓
Phase 8: Fix Loop (for each fixable issue)
    ├─ 8a: Locate source (grep + glob)
    ├─ 8b: Fix (minimal change)
    ├─ 8c: Commit (one commit per fix)
    ├─ 8d: Re-test + before/after screenshots
    ├─ 8e: Classify (verified / best-effort / reverted)
    ├─ 8e.5: Regression test (if applicable)
    └─ 8f: Self-regulation (WTF-likelihood every 5 fixes)
    ↓
Phase 9: Final QA (re-run affected pages)
Phase 10: Report (local + project-scoped)
Phase 11: TODOS.md Update
```

### Key Design Patterns

**Diff-Aware Mode (Automatic):** When invoked on a feature branch without a URL:
1. Analyzes branch diff to identify affected pages/routes
2. Maps file changes to their URLs (controllers → routes, components → pages)
3. Auto-detects running app on common ports (3000, 4000, 8080)
4. Tests only affected pages, scoped to branch changes

**Health Score Rubric:**
- Weighted categories: Console (15%), Links (10%), Visual (10%), Functional (20%), UX (15%), Performance (10%), Content (5%), Accessibility (15%)
- Per-category scoring: Critical=-25, High=-15, Medium=-8, Low=-3
- Health score = weighted average of category scores

**Test Framework Bootstrap:** If no test infrastructure detected:
1. Detect runtime (Node/Ruby/Python/Go/Rust/PHP/Elixir)
2. Research best practices for runtime
3. Ask user to choose framework
4. Install, configure, create example test
5. Generate 3-5 real tests for existing code
6. Set up CI/CD pipeline (GitHub Actions)
7. Create TESTING.md + update CLAUDE.md

**WTF-Likelihood Self-Regulation:**
```
Start: 0%
+15% per revert
+5% if fix touches >3 files
+1% per additional fix after 15
+10% if all remaining are Low severity
+20% if touching unrelated files
```
Hard cap: 50 fixes regardless.

**Fix Loop Atomicity:** Each fix is:
1. One commit per issue (never bundle)
2. One issue per iteration
3. Immediate re-test with before/after evidence
4. Regression test if testable

### Notable Clever Solutions

1. **Incremental documentation** — Issues written immediately when found, not batched. Captures state while it's fresh.

2. **Regression test classification** — Distinguishes verified (re-test confirms), best-effort (can't fully verify), reverted (caused regression), deferred (can't fix from source).

3. **Two evidence tiers** — Interactive bugs get step-by-step screenshots + snapshot diff. Static bugs get single annotated screenshot.

4. **Framework-specific guidance** — Next.js (hydration errors, _next/data requests), Rails (N+1, CSRF, Turbo/Stimulus), WordPress (plugin conflicts, mixed content), SPA (stale state, back button, memory leaks).

5. **Clean working tree requirement** — Forces commit/stash before QA starts. Each bug fix gets its own atomic commit with proper attribution.

### Edge Cases / Error Handling

- Dirty working tree: AskUserQuestion (commit/stash/abort)
- No browse binary: One-time build via setup script
- CAPTCHA blocking: Pause and ask user
- 2FA required: AskUserQuestion for code
- No local app found: Check staging/preview URL, then ask user
- Bootstrap fails: Revert all changes, warn and continue without tests
- Test framework already exists: Skip bootstrap, read conventions from 2-3 existing test files
- Regression test exploration >2 min: Skip and defer

### Technical Debt / Shortcuts

1. **Viewport checks are manual** — Only runs at 375x812 and 1280x720. Doesn't test full range of viewport sizes.

2. **WTF-likelihood is a heuristic** — No formal model. Stops at 50 fixes hard cap regardless of remaining issue severity.

3. **No test framework = no regression tests** — If user declined bootstrap and no framework exists, regression tests are skipped entirely.

4. **Browse binary path assumption** — Checks `~/.claude/skills/gstack/browse/dist/browse` first, then local `.claude/skills/`. Platform detection is implicit.

---

## Feature 3: Ship (Release Engineer)

### Purpose
Release engineer workflow. Syncs main, runs tests, audits coverage, pushes, and opens PR. Bootstraps test frameworks if none exist. Fully automated (non-interactive by design) — user says `/ship`, next thing they see is the PR URL.

### Core Implementation Files
- `ship/SKILL.md` (1,828 lines) — Main skill definition (largest skill in the repo)
- `bin/gstack-review-read` — Reads review log for dashboard
- `bin/gstack-review-log` — Persists review results
- `bin/gstack-diff-scope` — Categorizes diff by file type (frontend/backend/prompts/tests/docs/config)

### Workflow Flow

```
Step 0: Detect platform + base branch
    ↓
Step 1: Pre-flight
    ├─ Check current branch (abort if on base)
    ├─ Git status + diff stat
    ├─ Review Readiness Dashboard (from gstack-review-read)
    └─ Distribution pipeline check (if new artifact added)
    ↓
Step 1.5: Distribution Pipeline Check
    ├─ Detect new CLI binary / library / tool
    └─ Verify release workflow exists (or ask)
    ↓
Step 2: Merge base branch (BEFORE tests)
    └─ git fetch + git merge (auto-resolve simple conflicts)
    ↓
Step 2.5: Test Framework Bootstrap
    └─ (Same bootstrap as /qa — installs test framework, CI pipeline, TESTING.md)
    ↓
Step 3: Run tests (on merged code)
    ├─ Parallel: bin/test-lane + npm run test
    └─ Test Failure Ownership Triage:
        ├─ Classify each failure (in-branch vs pre-existing)
        ├─ In-branch: STOP
        └─ Pre-existing: solo (fix now/TODO/skip) or collaborative (blame+assign/TODO/skip)
    ↓
Step 3.25: Eval Suites (conditional)
    └─ If prompt-related files changed, run affected eval suites at full tier
    ↓
Step 3.4: Test Coverage Audit
    ├─ Same ASCII coverage diagram as /review
    ├─ Generate tests for gaps
    ├─ Coverage gate: Minimum (default 60%) / Target (default 80%)
    └─ Test plan artifact written for /qa consumption
    ↓
Step 3.45: Plan Completion Audit
    ├─ Discover plan file (same 3-tier as /review)
    ├─ Extract actionable items
    └─ Cross-reference against diff: DONE/PARTIAL/NOT DONE/CHANGED
    ↓
Step 3.47: Plan Verification
    ├─ Check for verification section in plan
    ├─ Detect running dev server
    └─ Invoke /qa-only inline (scoped to plan verification items)
    ↓
Step 3.5: Pre-Landing Review
    ├─ Same review checklist as /review
    ├─ Design review (conditional)
    └─ Adversarial review (auto-scaled, same as /review)
    ↓
Step 3.75: Greptile Comment Resolution (if PR exists)
    └─ Same Greptile integration as /review
    ↓
Step 3.8: Adversarial Review (auto-scaled)
    └─ Same as /review Step 5.7
    ↓
Step 4: Version bump (auto-decide)
    ├─ MICRO: <50 lines, trivial tweaks
    ├─ PATCH: 50+ lines, bug fixes
    ├─ MINOR: ASK (major features)
    └─ MAJOR: ASK (milestones)
    ↓
Step 5: CHANGELOG (auto-generate)
    ├─ Read all commits on branch
    ├─ Read full diff
    └─ Categorize: Added/Changed/Fixed/Removed
    ↓
Step 5.5: TODOS.md (auto-update)
    ├─ Check structure (skill/component grouping, P0-P4, Completed section)
    ├─ Detect completed TODOs from diff
    └─ Move completed to Done section
    ↓
Step 6: Commit (bisectable chunks)
    ├─ Infrastructure first (migrations, config)
    ├─ Models & services
    ├─ Controllers & views
    └─ VERSION + CHANGELOG + TODOS last
    ↓
Step 6.5: Verification Gate
    ├─ Re-run tests if code changed after Step 3
    └─ Run build if exists
    ↓
Step 7: Push
    └─ git push -u origin <branch>
    ↓
Step 8: Create PR/MR
    └─ PR body with: Summary, Test Coverage, Pre-Landing Review, Design Review, Eval Results, Greptile Review, Plan Completion, Verification Results, TODOS, Test plan
    ↓
Step 8.5: Auto-invoke /document-release
    └─ Sync docs, commit if changed, push
    ↓
Step 8.75: Persist ship metrics
    └─ Log coverage, plan completion, verification result to ~/.gstack/projects/{slug}/
```

### Key Design Patterns

**Non-Interactive Automation:** The defining characteristic. User says `/ship`, workflow runs straight through. Only stops for:
- On base branch (abort)
- Merge conflicts
- Test failures (in-branch)
- Version bump (MINOR/MAJOR — must ask)
- Greptile comments needing user decision
- Coverage below minimum (hard gate with override)
- Plan items NOT DONE (ask)

**Review Readiness Dashboard Integration:** Checks if Eng Review was run within 7 days. Required for shipping (can be disabled via `gstack-config set skip_eng_review true`).

**Coverage Gate with User Override:**
- >= target: Pass automatically
- >= minimum, < target: Ask (generate more or accept risk)
- < minimum: Must generate tests or explicitly override
- Coverage undetermined: Skip gate silently

**Bisectable Commit Strategy:**
- Infrastructure (migrations, config, routes) first
- Models & services second
- Controllers & views third
- VERSION + CHANGELOG + TODOS last
- Each commit must be independently valid

**Plan Verification Inline:** Runs /qa-only scoped to the plan's verification section. Catches if the implementation actually works before pushing.

**Auto-Document-Release:** After PR creation, automatically invokes `/document-release` to sync README, ARCHITECTURE, CLAUDE.md, TODOS. Commits and pushes if changes exist.

### Notable Clever Solutions

1. **Test Failure Ownership Triage** — Distinguishes in-branch vs pre-existing failures. In-branch = STOP (must fix). Pre-existing = depends on REPO_MODE (solo = fix now, collaborative = blame+assign). Prevents shipping broken tests while not blocking on old failures.

2. **Version Bump Auto-Decision** — `<50 lines = MICRO`, `50+ lines = PATCH`, `major features = MINOR (ask)`, `milestones = MAJOR (ask)`. No user input for common cases.

3. **4-Digit Version Format** — `MAJOR.MINOR.PATCH.MICRO` allows micro-bumps for trivial changes without consuming PATCH.

4. **Eval Tier Selection** — `/ship` always uses `full` tier (Opus persona judges) since it's a pre-merge gate. Clear decision: gate = highest quality.

5. **Ship Metrics Persistence** — Logs coverage %, plan items, verification result, version, branch to `~/.gstack/projects/{slug}/{branch}-reviews.jsonl` for `/retro` trend analysis.

### Edge Cases / Error Handling

- Merge conflicts: Try auto-resolve (VERSION, schema.rb, CHANGELOG ordering). If complex, STOP and show conflicts.
- No PR exists: Skip Greptile step. `/ship` creates the PR in Step 8.
- No dev server for plan verification: Skip silently (don't block).
- No verification section in plan: Skip silently.
- Coverage undetermined: Skip gate, don't default to 0%.
- Plan file found but irrelevant: Treat as "no plan file."
- All tests pass: Note briefly, continue silently.

### Technical Debt / Shortcuts

1. **Eval suite sequential execution** — If multiple eval suites need to run, they run sequentially (each needs its own test lane). If first fails, stops immediately — burns cost on remaining but avoids false confidence.

2. **No rollback on version bump** — If user overrides coverage gate and ships at low coverage, there's no automatic mechanism to revert the version bump if later tests fail.

3. **Greptile replies can fail silently** — If `gh api` reply fails, warns and continues. Comment remains unresolved but workflow doesn't block.

4. **Distribution pipeline check is simplistic** — Detects new artifacts by file pattern (`cmd/.*/main\.go`, `bin/`, `Cargo.toml`, `setup.py`, `package.json`). Doesn't detect new microservice that needs deployment pipeline.

5. **Auto-resolve of merge conflicts is conservative** — Only handles VERSION, schema.rb, CHANGELOG ordering. Other conflict types STOP the workflow rather than attempting resolution.

### Cross-Feature Integration

Ship is the convergence point for all review skills:
- `/ship` invokes `/review` (Step 3.5) for pre-landing review
- `/ship` invokes `/qa-only` (Step 3.47) for plan verification
- `/ship` invokes `/document-release` (Step 8.5) for doc sync
- `/ship` reads review log (Step 1) from all prior review runs
- `/ship` persists metrics to `reviews.jsonl` for `/retro` consumption

---

## Cross-Feature Observations

### Shared Infrastructure
- `gstack-review-read` and `gstack-review-log` are shared between `/review` and `/ship`
- `gstack-diff-scope` is used by `/review` and `/ship`
- `gstack-slug` is used by all three for project identification
- Test Framework Bootstrap is nearly identical between `/qa` and `/ship`

### Shared Patterns
- **Plan file discovery** is identical in `/review` (Step 1.5) and `/ship` (Step 3.45)
- **Scope drift detection** uses the same cross-reference logic
- **Fix-First Heuristic** is shared reference (checklist.md)
- **WTF-likelihood self-regulation** appears in `/qa` (8f) but not `/ship` — `/ship` relies on coverage gate instead
- **Adversarial review auto-scaling** is identical in `/review` and `/ship`

### Discrepancies / Inconsistencies

1. **Greptile suppression history format differs** — `/review` and `/ship` both write to `greptile-history.md` but the format doesn't include the exact grep patterns used for suppression matching (category-based only).

2. **WTF-likelihood only in /qa** — `/ship` has no equivalent self-regulation mechanism. The 50-fix hard cap in `/qa` has no counterpart in `/ship`.

3. **Coverage diagram is ASCII in both but...** — `/ship` writes the test plan artifact for `/qa` consumption, but there's no programmatic way for `/qa` to read the ASCII coverage diagram from `/ship`.

4. **Test Framework Bootstrap is duplicated** — Nearly identical code in `/qa` (Phase 2 bootstrap) and `/ship` (Step 2.5). Could be extracted to a shared reference.

### Notable Technical Decisions

1. **Non-interactive by default** — `/ship` is explicitly designed to run without asking. Contrast with `/review` and `/qa` which frequently use `AskUserQuestion`. This is a deliberate architectural choice: shipping should be a pipeline, not a conversation.

2. **Review log at project level** — Reviews are stored in `~/.gstack/projects/{slug}/{branch}-reviews.jsonl`, not in the repo itself. This allows cross-repo analysis in `/retro global` but means review history is tied to the developer's machine.

3. **Bash for tooling** — All helper scripts (`gstack-review-read`, `gstack-diff-scope`, etc.) are Bash, not TypeScript or a unified CLI. This keeps dependencies minimal but makes the tooling less discoverable as a coherent system.

4. **SKILL.md as source of truth** — The skill definition IS the SKILL.md file. Templates (`.tmpl`) generate the SKILL.md files. This means the human-readable skill definition is also the machine-executable artifact. Unusual but pragmatic.
