# gstack Feature Deep Dive

**Repository:** `/Users/sheldon/Documents/claw/reference/gstack`
**Synthesized:** 2026-03-27
**Source:** Feature index + 9 batch research documents
**Output:** `/Users/sheldon/Documents/claw/gstack-study/learning/04-features-deep-dive.md`

---

## Executive Summary

gstack is an AI-augmented development workflow system that transforms Claude Code into a virtual engineering team. It provides 28 specialized skills organized around the sprint lifecycle: Think, Plan, Build, Review, Test, Ship, Reflect.

The architecture follows a consistent pattern: skills are defined as large prompt templates (SKILL.md files, auto-generated from .tmpl source templates via `bun run gen:skill-docs`), with shared infrastructure components (preamble system, browse daemon, review dashboard, telemetry). The browse daemon is the primary external system dependency, providing persistent Chromium state for QA, canary, and benchmark skills.

**Core insight:** gstack is not a code pipeline -- it is a **judgment pipeline**. Most skills do not execute code; they produce structured analysis, decisions, and recommendations that guide human and AI action. The shipping skills (/ship, /land-and-deploy) are the exception, orchestrating real git/GitHub operations end-to-end.

---

## Tier 1: Core Workflow Skills

These five skills form the primary sprint workflow, roughly mapping to the sequence: think deeply (office-hours) -> scope (plan-ceo-review) -> architect (plan-eng-review) -> ship (ship) -> verify (review, qa).

---

### 1. Office Hours (`/office-hours`)

**What it does:** YC-style startup diagnostic session. Two modes: startup (six forcing questions routed by product stage) and builder (generative design thinking for side projects).

**Workflow pipeline:**
- Phase 1: Context gathering (CLAUDE.md, TODOS.md, git log/diff, codebase mapping)
- Phase 2A/2B: Mode-specific questioning (startup or builder)
- Phase 2.75: Landscape awareness via web search with three-layer synthesis
- Phase 3: Premise challenge (right problem? what if nothing done?)
- Phase 4: Alternatives generation (mandatory: minimal viable + ideal architecture)
- Design sketch phase (if UI): wireframe HTML via headless browser
- Phase 5: Design doc written to `~/.gstack/projects/{slug}/`
- Phase 6: Three-beat handoff with Garry's YC pitch plea

**Key design decisions:**
- Smart-skip routing: doesn't ask all six questions, routes based on product stage (pre-product, has users, has paying customers, pure infra)
- Anti-sycophancy rules: explicit list of phrases never to say
- Founder signal synthesis: tracks 8 founder qualities, uses count to tier the closing
- Design lineage: `Supersedes` field creates revision chain across sessions
- Hard gate: "Do NOT invoke any implementation skill, write any code"

**Technical debt:**
- No hard state between questions (mid-session context loss)
- Design doc stored in home dir, not in project repo
- Phase 2.75 always runs even for very specific technical problems

---

### 2. Plan CEO Review (`/plan-ceo-review`)

**What it does:** CEO/founder-mode plan review. Rethinks the problem to find the 10-star product. Four modes: SCOPE EXPANSION, SELECTIVE EXPANSION, HOLD SCOPE, SCOPE REDUCTION.

**Workflow pipeline:**
- Pre-review: git log/diff/stash, grep for TODO/FIXME/HACK, read CLAUDE.md/TODOS.md, check for prior design doc and CEO handoff notes
- Step 0A-0F: Nuclear scope challenge (premise, existing leverage, dream state, alternatives, mode-specific analysis, mode selection)
- 10 review sections: Architecture, Error & Rescue Map, Security, Data Flow, Code Quality, Test Review, Performance, Observability, Deployment, Long-Term Trajectory (+ Design if UI scope)
- Outside voice: optional Codex or Claude subagent
- CEO Plan persistence: writes to `~/.gstack/projects/{slug}/ceo-plans/`

**Key design decisions:**
- Four-mode framework with 8-column quick reference
- "Prime Directives": zero silent failures, every error has a name, data flows have shadow paths
- 6 mandatory ASCII diagrams
- Cross-model tension tracking: logs where Claude and Codex disagree

**Technical debt:**
- Complexity check threshold ("8 files or 2 new classes") is arbitrary
- Manual diagram maintenance (no stale detection)
- 10 review sections is aggressive for some plans

---

### 3. Plan Engineering Review (`/plan-eng-review`)

**What it does:** Engineering manager-mode plan review. Locks architecture, data flow, edge cases, and tests. Forces hidden assumptions into the open.

**Workflow pipeline:**
- Design doc check (looks for `~/.gstack/projects/{slug}/...-design-*.md`)
- Step 0: Scope challenge (7 checks including complexity, search, distribution)
- 4 review sections: Architecture, Code Quality, Test Review (highest detail), Performance
- Test plan artifact written to `~/.gstack/projects/{slug}/` -- consumed by `/qa`

**Key design decisions:**
- Test coverage diagram format shows both code path AND user flow coverage
- Quality scoring rubric: ★★★ (edge cases + error paths) vs ★★ (happy path) vs ★ (smoke test)
- E2E decision matrix: explicit criteria for when E2E is right tool vs unit
- Regression rule with IRON RULE: regressions must have regression tests
- "Tests are cheapest lake to boil": anti-deferral argument

**Technical debt:**
- Coverage diagram is aspirational (no automated verification)
- Auto-detect test framework could be wrong for monorepos

---

### 4. Review (`/review`)

**What it does:** Staff engineer PR review. Finds bugs that pass CI but blow up in production. Auto-fixes obvious issues, flags completeness gaps.

**Workflow pipeline:**
- Step 1.5: Scope drift detection (TODOS.md vs PR description vs commits vs diff)
- Step 2.5: Greptile comment fetch (if PR exists) with suppression history
- Two-pass review: Pass 1 (CRITICAL: SQL safety, race conditions, LLM trust boundary, enum completeness) + Pass 2 (INFORMATIONAL: 7 categories)
- Step 4.75: Test coverage diagram
- Fix-first review: AUTO-FIX items applied immediately, ASK items batched for user approval
- Step 5.7: Adversarial review auto-scaled by diff size (small/medium/large)

**Key design decisions:**
- Fix-first heuristic: mechanical fixes auto-applied, complex issues require judgment
- Plan file discovery: 3-tier fallback (conversation context -> content grep -> recent 24h)
- Greptile integration: two-tier reply system, per-project and global suppression
- Scope drift detection: NOT DONE items become MISSING REQUIREMENTS evidence

**Technical debt:**
- Fix-first without revert path (auto-fix regression = manual catch)
- Coverage diagram is ASCII text (not programmatically parseable)
- Greptile position=null filter silently drops comments on deleted lines

---

### 5. QA (`/qa`)

**What it does:** QA Lead that tests the app, finds bugs, fixes them with atomic commits, and auto-generates regression tests.

**Workflow pipeline:**
- Phase 2: Authenticate (cookie import OR form fill OR 2FA)
- Phase 3: Orient (snapshot, annotated screenshot, map links, console errors)
- Phase 4: Explore (systematic page visit, depth judgment per feature)
- Phase 8: Fix loop (8a-8f: locate source, fix, commit, re-test, classify, regression test, self-regulation)
- Phase 10: Report (local + project-scoped)

**Key design decisions:**
- Diff-aware mode: analyzes branch diff to identify affected pages, maps file changes to URLs
- Health score rubric: weighted categories (Console 15%, Links 10%, Visual 10%, Functional 20%, UX 15%, Performance 10%, Content 5%, Accessibility 15%)
- WTF-likelihood self-regulation: starts 0%, +15% per revert, +5% if fix touches >3 files, hard cap 50 fixes
- Fix loop atomicity: one commit per issue, immediate re-test with before/after evidence
- Test framework bootstrap: detects runtime, installs framework, creates CI pipeline, generates TESTING.md

**Technical debt:**
- Viewport checks only at 375x812 and 1280x720
- No test framework = no regression tests (if user declined bootstrap)
- WTF-likelihood is a heuristic with no formal model

---

### 6. Ship (`/ship`)

**What it does:** Release engineer workflow. Syncs main, runs tests, audits coverage, pushes, opens PR. Fully non-interactive by design.

**Workflow pipeline:**
- Step 2: Merge base branch (before tests, auto-resolve simple conflicts)
- Step 2.5: Test framework bootstrap (same as /qa)
- Step 3: Run tests with failure ownership triage (in-branch = STOP, pre-existing = solo fix/TODO/skip or collaborative blame)
- Step 3.25: Eval suites (conditional, prompt-related files only)
- Step 3.4: Coverage audit with ASCII diagram + test plan artifact for /qa
- Step 3.45: Plan completion audit (DONE/PARTIAL/NOT DONE/CHANGED cross-reference)
- Step 3.47: Plan verification (invokes /qa-only inline scoped to plan verification)
- Step 3.5: Pre-landing review (same as /review)
- Step 3.75: Greptile comment resolution
- Step 4: Version bump auto-decide (MICRO/<50 lines, PATCH/50+ lines, MINOR/MAJOR = ask)
- Step 5: CHANGELOG auto-generate
- Step 5.5: TODOS.md auto-update
- Step 6: Bisectable commit strategy (infra first, models/services, controllers/views, VERSION/CHANGELOG/TODOS last)
- Step 8.5: Auto-invoke /document-release

**Key design decisions:**
- Non-interactive by default: only stops for merge conflicts, test failures, version bump, coverage gate, plan items not done
- Review readiness dashboard: checks if Eng Review ran within 7 days
- Coverage gate with override: >= target = pass, >= minimum = ask, < minimum = must generate or override
- Bisectable commits: each must be independently valid
- Ship metrics persistence: logs coverage, plan items, verification to `~/.gstack/projects/{slug}/`

**Technical debt:**
- Eval suite sequential execution (stops on first failure)
- No rollback on version bump override
- Auto-resolve of merge conflicts conservative (only VERSION, schema.rb, CHANGELOG ordering)

---

### 7. Browse (`$B`, `/browse`)

**What it does:** Persistent Chromium daemon with ~100ms per command latency. State (cookies, tabs, localStorage) persists across commands.

**Architecture:**

```
CLI (browse/src/cli.ts)
  └── reads .gstack/browse.json for {pid, port, token}
  └── ensures server running (health check)
  └── HTTP POST to http://127.0.0.1:{port}/command
  └── prints response to stdout

Server (browse/src/server.ts)
  └── Bun.serve HTTP on random port (10000-60000)
  └── routes: READ_COMMANDS, WRITE_COMMANDS, META_COMMANDS

BrowserManager (browse/src/browser-manager.ts)
  └── Playwright chromium.launch() -- single browser instance
  └── manages multiple tabs (Map<tabId, Page>)
  └── dialog auto-handling
  └── state save/restore for handoff and useragent changes
```

**Key subsystems:**

**Snapshot with ref system (`snapshot.ts`):**
- Uses Playwright's `page.ariaSnapshot()` for accessibility tree
- Assigns `@e1`, `@e2`, ... refs to interactive elements
- Stores `Map<ref, {locator, role, name}>` on BrowserManager
- Locators are live references (re-resolve on every call, no stale DOM failures)
- `-D` diff mode: stores raw text, uses `diff` npm package for unified diffs
- `-C` cursor-interactive scan: falls back to JS evaluation for pointer elements
- `-a` annotated screenshot: injects overlay divs at each ref's bounding box

**Circular buffers (`buffers.ts`):**
- O(1) insert ring buffers with 50,000 entry capacity
- Console, network, dialog entries captured via Playwright page event listeners
- Flushed to disk every 1 second via setInterval (async writes don't affect latency)

**Browser lifecycle:**
- Launch: `chromium.launch({headless: true})` with platform-specific handling (--no-sandbox for container/CI, disabled sandbox for Windows)
- Chromium crash: `browser.on('disconnected')` calls `process.exit(1)`, CLI auto-restarts on next command
- State save/restore: captures cookies + per-page URLs + localStorage/sessionStorage
- Handoff: launch headed browser, restore state, close old headless (fire-and-forget)

**CLI server lifecycle:**
- Lockfile: `O_CREAT | O_EXCL` atomic check-and-create (prevents concurrent startup races)
- Health check: HTTP GET /health (more reliable than PID check on Windows)
- Version mismatch: CLI compares binary hash vs server's stored `binaryVersion`, auto-restarts if different
- Connection loss recovery: retries once by restarting server on ECONNREFUSED/ECONNRESET

**Security:**
- Path validation: `eval <file>` and `cookie-import` validate absolute paths within TEMP_DIR or cwd
- Sensitive value redaction: storage command redacts tokens, API keys, JWTs by key name and value prefix
- Header redaction: sensitive headers redacted in success message

**Cookie import (`cookie-import-browser.ts`):**
- Supports Chrome, Arc, Brave, Edge, Comet on macOS and Linux
- Decryption: AES-128-CBC via Keychain (macOS) or libsecret (Linux)
- Profile display names: reads Chrome's Preferences JSON for account email
- SQLite locked handling: copies to /tmp with WAL/SHM, schedules cleanup

**Technical debt:**
- No Windows support for cookie import (keychain/DAC only)
- vendored = no git history (depth-1 clone loses all history)
- sed -i '' portability issue in gstack-config (macOS syntax)

---

## Tier 2: Specialized Skills

---

### 8. Plan Design Review (`/plan-design-review`)

**What it does:** Senior product designer who reviews a plan (not a live site). 7-pass structured review with 0-10 ratings per dimension.

**Workflow:** Pre-review audit -> 7 passes (Information Architecture, Interaction States, User Journey, AI Slop Risk, Design System Alignment, Responsive & Accessibility, Unresolved Decisions) -> completion summary with before/after ratings -> TODOS.md proposals

**AI slop detection:** 10 hard rejection patterns (purple gradients, 3-column icon grids, centered everything, bubbly border-radius, decorative blobs, emoji as design, colored left-border on cards, generic hero copy, cookie-cutter section rhythm, beautiful image with weak brand)

**Outside voices:** Codex + Claude subagent run in parallel, synthesized into litmus scorecard

**Key design decisions:**
- 9 explicit design principles (empty states are features, accessibility is not optional, subtraction default)
- 12 perceptual instincts that run automatically during review
- Hard rejections from outside voices pre-loaded as FIRST items in Pass 1

**Technical debt:**
- 7 passes are sequential with user pause after each
- No automated verification that design decisions fix issues raised

---

### 9. Design Consultation (`/design-consultation`)

**What it does:** Senior product designer who acts as design consultant. Builds complete design system from scratch. Proposes SAFE/RISK breakdowns, generates HTML preview page.

**Workflow:**
- Phase 0: Pre-checks (existing DESIGN.md? codebase context)
- Phase 1: Comprehensive question covering product, users, type, research preference
- Phase 2: Competitive research via WebSearch + browse (3-layer synthesis)
- Phase 3: Complete proposal with SAFE/RISK breakdown
- Phase 5: Font & color preview HTML page (self-contained, dogfoods the design system)
- Phase 6: Write DESIGN.md + update CLAUDE.md

**Design knowledge built in:**
- 10 aesthetic directions (Brutally Minimal, Maximalist Chaos, Retro-Futuristic, etc.)
- Font recommendations by purpose (display, body, data/tables, code)
- Font blacklist: Papyrus, Comic Sans, Lobster, Impact, etc.
- Overused fonts: Inter, Roboto, Arial, Helvetica, Open Sans, Lato, Montserrat, Poppins

**SAFE/RISK breakdown:** Every proposal has creative risks by design -- coherence is table stakes, the real question is where the product takes creative risks.

**Technical debt:**
- Research phase degrades gracefully but quality varies significantly
- No offline fallback for Google Fonts in preview page

---

### 10. Design Review (`/design-review`)

**What it does:** Senior product designer + frontend engineer who audits live sites and iteratively fixes what they find. Each fix committed atomically with before/after screenshot evidence.

**Workflow:** 11 phases: First Impression -> Design System Extraction -> Page-by-Page Visual Audit (10 categories) -> Interaction Flow Review -> Cross-Page Consistency -> Compile Report (Design Score A-F, AI Slop Score A-F) -> Triage -> Fix Loop -> Final Design Audit -> Report -> TODOS.md Update

**Scoring system:**
- Design Score: weighted average across 10 categories (Visual Hierarchy 15%, Typography 15%, Spacing 15%, Color/Contrast 10%, Interaction States 10%, Responsive 10%, Content 10%, AI Slop 5%, Motion 5%, Performance 5%)
- AI Slop Score: standalone A-F grade

**Self-regulation in fix loop:**
- Risk computed after every 5 fixes
- Start 0%, +15% per revert, +5% per JSX/TSX file touched, +20% for unrelated files
- Hard cap: 20% risk threshold triggers STOP, 30 fixes maximum

**Technical debt:**
- No tests for the skill itself
- AI slop source citation references GPT-5.4 (hypothetical future model)

---

### 11. QA Only (`/qa-only`)

**What it does:** QA reporter. Same methodology as /qa but report only without code changes.

**Key differences from /qa:**
- Never fixes bugs
- No regression test generation
- No test framework bootstrap
- Rule 5: "Never read source code. Test as a user, not a developer."
- Report-only sections (no "Fixes Applied" or "Regression Tests")

**Modes:** Diff-aware, Full, Quick (30-second smoke), Regression (compare to baseline)

**Technical debt:**
- No regression test infrastructure
- Screenshot accumulation in `.gstack/qa-reports/` (no cleanup)

---

### 12. CSO (`/cso`)

**What it does:** Security audit combining OWASP Top 10 assessment, STRIDE threat modeling, and active verification. Two modes: daily (8/10 confidence gate, zero-noise) and comprehensive (2/10 bar).

**15 phases:** Architecture mental model -> Attack surface census -> Secrets archaeology -> Dependency supply chain -> CI/CD pipeline security -> Infrastructure shadow surface -> Webhook audit -> LLM & AI security -> Skill supply chain -> OWASP Top 10 -> STRIDE threat model -> Data classification -> False positive filtering + active verification -> Findings report -> Save report

**False positive filtering (22 hard exclusions + 12 precedents):**
- EXCLUSIONS: DoS/resource exhaustion (except LLM cost amplification), secrets on disk if secured, input validation without proven impact, race conditions unless concretely exploitable, outdated lib CVEs with CVSS < 4.0, etc.
- PRECEDENTS: Logging secrets is vuln, logging URLs is safe; UUIDs are unguessable; React/Angular are XSS-safe by default

**Active verification:**
- For each finding surviving confidence gate: parallel Agent sub-task verification with 1-10 score
- Discard below threshold
- Variant analysis: search entire codebase for same pattern when finding is VERIFIED

**Skill supply chain (Phase 8):**
- Scans installed skills for malicious patterns (credential exfiltration, prompt injection)
- Tier 1: auto-scan repo-local skills
- Tier 2: requires user permission for globally installed skills
- Informed by Snyk ToxicSkills research (36% of AI agent skills have security flaws, 13.4% malicious)

**LLM & AI security (Phase 7):**
- Prompt injection vectors, unsanitized LLM output, tool calling without validation
- Key precedent: user content in user-message position is NOT prompt injection

**Technical debt:**
- Git history scanning bounded (misses rotated secrets, binary files)
- CI/CD analysis limited to YAML parsing (doesn't execute workflows)
- Diff mode history scanning only looks at current branch commits

---

### 13. Investigate (`/investigate`)

**What it does:** Systematic root-cause debugging. Iron Law: NO FIXES WITHOUT INVESTIGATION. Traces data flow, tests hypotheses, 3-strike rule before escalation.

**Key design decisions:**
- Scope lock: after forming hypothesis, locks edits to affected directory using `/freeze`
- 3-strike escalation: after 3 failed hypotheses, escalate to user
- Sanitized web search: strips IPs, paths, SQL, customer data before searching
- Pattern table: pre-defined bug signatures for structured starting point

**Technical debt:**
- No persistent investigation state between sessions
- Shell-based freeze check fragile if tool input format changes

---

### 14. Canary (`/canary`)

**What it does:** Post-deploy monitoring. Watches for console errors, performance regressions, page failures.

**Monitoring loop:** Every 60s: goto page, snapshot with annotated screenshot, console --errors, perf. Alert levels: CRITICAL (page load failure), HIGH (new console errors), MEDIUM (load time > 2x baseline), LOW (new 404s).

**Key design decisions:**
- Baseline comparison not absolutes: only NEW errors trigger alerts (prevents fatigue)
- Smart tolerance: patterns must persist across 2+ consecutive checks
- Auto page discovery: if --pages not specified, discovers top 5 nav links

**Technical debt:**
- Hardcoded 60s intervals (not configurable)
- 2x threshold is arbitrary
- No alert deduplication across monitoring session
- No background execution (stops if Claude disconnects)

---

### 15. Benchmark (`/benchmark`)

**What it does:** Performance regression detection. Baselines page load times, Core Web Vitals, resource sizes. Compares before/after on every PR.

**Metrics collected:**
- Navigation timing: TTFB, FCP, LCP (approximated), DOM Interactive, DOM Complete, Full Load
- Resource analysis: top 15 slowest resources, bundle sizes by type
- Network summary: total requests, total transfer size by type

**Regression thresholds:** Timing (>50% increase OR >500ms absolute), Bundle size (>25% increase), Request count (>30% increase warning)

**Key design decisions:**
- JavaScript instrumentation: uses `performance.getEntriesByType()` directly
- Resource attribution: clean filenames (removes path and query string)
- Third-party separation: flags non-actionable third-party scripts
- --diff mode: uses `gh pr view` to detect base branch, benchmarks only affected pages
- --trend mode: loads historical baselines, detects creeping regressions

**Technical debt:**
- No real Core Web Vitals (FCP/LCP approximated from paint entries)
- Single page load sample (no warm-up, no median/percentile)
- No network conditioning (doesn't simulate slow 3G)

---

## Tier 3: Infrastructure & Power Tools

---

### 16. Land and Deploy (`/land-and-deploy`)

**What it does:** Release engineer single command. Merges PR, waits for CI and deploy, verifies production health. One command from "approved" to "verified in production."

**Workflow:**
1. Pre-flight (auth check, parse args, detect PR)
2. Pre-merge checks (CI status, merge conflicts)
3. Wait for CI (15-min timeout)
3.5. Pre-merge readiness gate (reviews staleness, free tests, E2E tests, LLM judge evals, PR body accuracy, doc check)
4. Merge PR (--auto or --squash --delete-branch)
5. Deploy strategy detection (Fly.io, Render, Vercel, Netlify, Heroku, GitHub Actions)
6. Wait for deploy (platform-specific polling, 20-min timeout)
7. Canary verification (conditional depth based on diff-scope)
8. Revert if needed
9. Deploy report (ASCII summary + JSONL log)
10. Suggest follow-ups

**Pre-merge readiness gate (Step 3.5):**
- Reviews staleness: compares review commit vs current HEAD, flags if 4+ commits since review
- Free tests: runs `bun test` immediately
- E2E tests: checks recent eval results from `~/.gstack-dev/evals/`
- LLM judge evals: checks for today's results
- PR body accuracy: compares against actual commits
- Document-release check: warns if docs not updated despite new features

**Canary depth by diff-scope:**
- SCOPE_DOCS only: skip entirely
- SCOPE_CONFIG only: smoke only ($B goto + 200 status)
- SCOPE_BACKEND only: console errors + perf check
- SCOPE_FRONTEND (any): full suite (console + perf + screenshot)
- Mixed: full canary

**Technical debt:**
- GitLab explicitly unsupported
- No retry on transient failures
- Merge conflict resolution not automated
- 30-min merge queue timeout is long
- No platform-specific deploy timeout (uses generic 20-min)

---

### 17. Autoplan (`/autoplan`)

**What it does:** Automated review pipeline. Runs CEO, design, and eng review automatically with encoded decision principles. Surfaces only taste decisions for approval.

**Workflow:**
- Phase 0: Intake + restore point capture (backs up plan file)
- Phase 1: CEO Review (reads plan-ceo-review/SKILL.md from disk, runs at full depth)
- Phase 2: Design Review (conditional on UI scope, reads plan-design-review/SKILL.md)
- Phase 3: Eng Review + Dual Voices (reads plan-eng-review/SKILL.md, dual Claude subagent + Codex)
- Phase 4: Final approval gate (N taste decisions, M auto-decided)
- Write review logs

**The 6 decision principles:**
1. Choose completeness
2. Boil lakes (fix everything in blast radius < 1 day)
3. Pragmatic (cleaner solution when same fix)
4. DRY
5. Explicit over clever
6. Bias toward action

**Dual voices architecture:**
- Claude subagent (via Agent tool): independent Claude instance, no prior-phase context
- Codex (via Bash): `codex exec` with read-only + web search cached
- Degradation matrix: both fail -> single-reviewer warning; one fail -> tagged; both succeed -> consensus table

**Key design decisions:**
- Skill file loading from disk: updates to plan-ceo-review etc. automatically incorporated
- Sequential phases that build: CEO -> Design -> Eng
- Decision audit trail: every auto-decision logged to plan file incrementally
- Pre-gate verification checklist: forces completion of all required outputs

**Technical debt:**
- Skip list for loaded skills could drift out of sync
- No cancellation support
- Test plan file path complexity (user/branch with special chars)

---

### 18. Document Release (`/document-release`)

**What it does:** Technical writer skill. Updates all project docs to match shipped changes. Polishes CHANGELOG voice, cleans up TODOS, optionally bumps VERSION.

**Workflow:**
- Step 1: Pre-flight & diff analysis (classify: new features, changed behavior, removed, infrastructure)
- Step 2: Per-file audit (README, ARCHITECTURE, CONTRIBUTING, CLAUDE.md, other .md)
- Step 3: Apply auto-updates (factual corrections via Edit tool)
- Step 4: Ask about risky changes (narrative, removals, rewrites >10 lines)
- Step 5: CHANGELOG voice polish (never clobber, polish wording)
- Step 6: Cross-doc consistency check
- Step 7: TODOS.md cleanup (mark completed, scan for TODO/FIXME/HACK comments)
- Step 8: VERSION bump question
- Step 9: Commit & output

**What NEVER happens:**
- CHANGELOG entries overwritten or regenerated
- Write tool on CHANGELOG.md (always Edit with exact old_string)
- Bump VERSION without asking
- Auto-update narrative sections
- Remove entire sections

**CHANGELOG voice rules:** Lead with what user can DO, not implementation details. "You can now..." not "Refactored..."

**Key design decisions:**
- "A real incident occurred where an agent replaced existing CHANGELOG entries" -- encoded as permanent safeguard
- VERSION bump scope verification: if already bumped, checks whether CHANGELOG covers ALL changes
- PR/MR body update is idempotent and race-safe

**Technical debt:**
- Maxdepth 2 for doc discovery (misses deeply nested)
- No automated link validation
- CONTRIBUTING.md smoke test is walk-through only (doesn't run commands)

---

### 19. Retro (`/retro`)

**What it does:** Team-aware weekly retro. Per-person breakdowns, shipping streaks, test health trends, growth opportunities. Supports `/retro global` across all projects.

**Data flow:** `git log` -> commit parser -> metrics computer -> narrative generator

**Git analysis pipeline (12 parallel commands):**
- Commits with stats, per-commit LOC breakdown (test vs prod), timestamps for session detection
- Files changed for hotspot analysis, PR/MR numbers, per-author file hotspots
- Per-author commit counts, Greptile signal, TODOS.md backlog health
- Regression test commits, skill usage telemetry

**Session detection:** 45-minute gap threshold between consecutive commits. Deep sessions (50+ min), medium (20-50), micro (<20).

**Midnight alignment:** Day/week windows are midnight-aligned to avoid timezone drift.

**Global retrospective mode:** Uses `gstack-global-discover` to find all AI coding sessions across repos, aggregates personal stats, saves to `~/.gstack/retros/`.

**Technical debt:**
- No caching of git queries (every retro re-fetches)
- TODOS.md parsing fragile (relies on `## Completed` section)
- No handling of bare repos without origin

---

### 20. Setup Browser Cookies (`/setup-browser-cookies`)

**What it does:** Session manager that imports cookies from real browser into headless session for authenticated page testing.

**Architecture:**
```
cookie-picker-ui.ts     -> HTML/CSS/JS (runs in user's real browser)
cookie-picker-routes.ts -> HTTP API (runs in browse server)
cookie-import-browser.ts -> Decryption engine (pure logic, no HTTP)
```

**Supported browsers:** Comet, Chrome, Chromium, Arc, Brave, Edge (macOS + Linux)

**Decryption pipeline:** Cookie DB (SQLite) -> Decrypt (AES-128-CBC via Keychain/libsecret) -> Playwright Cookie -> page.context().addCookies()

**Key security considerations:**
- Key cache: derived AES keys cached per browser to avoid repeated Keychain lookups
- Profile validation: rejects path traversal (`../`) and control characters
- SQL injection prevention: parameterized queries throughout
- No shell interpolation: browser names go through registry lookup

**Technical debt:**
- No Windows support
- No parallel decryption (sequential for 100+ cookies)
- Keychain timeout hardcoded 10s (no retry with longer timeout)
- importedDomains is in-memory (lost on server restart)

---

### 21. Setup Deploy (`/setup-deploy`)

**What it does:** One-time setup for /land-and-deploy. Detects platform, production URL, deploy commands automatically.

**Platform detection:** fly.toml (Fly.io), render.yaml (Render), vercel.json/.vercel (Vercel), netlify.toml (Netlify), Procfile (Heroku), railway.json/toml (Railway), GitHub Actions workflows

**Configuration written to CLAUDE.md:**
- Platform, production URL, deploy workflow, deploy status command, merge method, project type, post-deploy health check
- Custom deploy hooks: pre-merge, deploy trigger, deploy status, health check

**Idempotency:** designed to be run multiple times safely (offers reconfigure/edit/keep)

**Technical debt:**
- Single deploy target only
- No rollback config
- No staging environment
- No secrets management integration

---

### 22. Codex (`/codex`)

**What it does:** Independent code review from OpenAI Codex CLI. Three modes: review (pass/fail gate), challenge (adversarial), consult (ask anything with session continuity).

**Review mode:** `codex review --base <base> -c 'model_reasoning_effort="xhigh"' --enable web_search_cached`. Gate verdict: FAIL if [P1] markers present.

**Challenge mode:** Uses `codex exec` with JSONL streaming, parses `[codex thinking]` reasoning traces.

**Consult mode:** Session continuity via `thread.started` event's thread_id, saves to `.context/codex-session-id`.

**Key technical decisions:**
- No model hardcoding: uses Codex's current default (automatically benefits from OpenAI improvements)
- Maximum reasoning effort: all modes use `model_reasoning_effort="xhigh"`
- Read-only sandbox: `-s read-only` flag
- 5-minute timeout on all bash calls

**Cross-model comparison:** When both `/review` (Claude) and `/codex review` have run, produces overlap analysis.

**Technical debt:**
- Cost estimation is token-based but actual cost varies
- Session resume failure gracefully degrades but loses conversation context

---

## Tier 4: Safety & Utility Tools

---

### 23. Careful (`/careful`)

**What it does:** Warns before destructive bash commands. Active for session duration. User can override each warning.

**Destructive patterns protected:**
- `rm -rf`, `rm -r`, `rm --recursive`
- `DROP TABLE`, `DROP DATABASE`, `TRUNCATE`
- `git push --force` / `-f`
- `git reset --hard`
- `git checkout .` / `git restore .`
- `kubectl delete`
- `docker rm -f` / `docker system prune`

**Safe exceptions:** `rm -rf node_modules` / `.next` / `dist` / `__pycache__` / `.cache` / `build` / `.turbo` / `coverage`

**Return value:** `{"permissionDecision":"ask"}` to warn, `{}` to allow.

**Technical debt:**
- Single-pattern detection (only sets one warning even if multiple match)
- No protection against rsync --delete, dd, shred, terraform destroy, az vm delete
- `--recursive` flag detection imprecise (`rm -fr` not matched)

---

### 24. Freeze (`/freeze`)

**What it does:** Restricts Edit and Write operations to a specific directory. Blocks outside boundary (not warns).

**Return value:** `{"permissionDecision":"deny"}` if outside, `{}` if inside.

**Trailing slash security:** `/src/` only matches `/src/`, not `/src-old`. Prevents prefix-matching bypass.

**What is NOT protected:** Read, Bash, Glob, Grep, and other tools. Bash commands like `sed` can still modify files outside boundary.

**Technical debt:**
- No protection against Bash with sed/echo/tee
- No directory-level protection for Read
- Symlink traversal could bypass

---

### 25. Guard (`/guard`)

**What it does:** Combines `/careful` and `/freeze` into single command.

**Implementation:** Composes sibling skill hook scripts:
```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      command: "bash ${CLAUDE_SKILL_DIR}/../careful/bin/check-careful.sh"
    - matcher: "Edit"
      command: "bash ${CLAUDE_SKILL_DIR}/../freeze/bin/check-freeze.sh"
    - matcher: "Write"
      command: "bash ${CLAUDE_SKILL_DIR}/../freeze/bin/check-freeze.sh"
```

**Technical debt:**
- Silent failure if sibling skills not installed
- Hook scripts remain registered after /unfreeze (just allow everything since no state file)

---

### 26. Unfreeze (`/unfreeze`)

**What it does:** Removes the freeze boundary set by `/freeze`.

**Implementation:** Deletes `~/.gstack/freeze-dir.txt`, reports previous boundary before deletion.

**Note:** Reports what the boundary was (helpful confirmation) before clearing.

---

### 27. Gstack Upgrade (`/gstack-upgrade`)

**What it does:** Self-updater that upgrades gstack to latest version. Handles 6 install types (global-git, local-git, vendored variants), syncs local copies, shows changelog.

**Install type detection (priority order):**
1. `$HOME/.claude/skills/gstack/.git` -> global-git
2. `$HOME/.gstack/repos/gstack/.git` -> global-git
3. `.claude/skills/gstack/.git` -> local-git
4. `.agents/skills/gstack/.git` -> local-git
5. `.claude/skills/gstack` (no .git) -> vendored
6. `$HOME/.claude/skills/gstack` (no .git) -> vendored-global

**Upgrade by type:**
- Git: `git stash -> git fetch -> git reset --hard origin/main -> ./setup`
- Vendored: fresh `git clone --depth 1` to temp, swap with old, run setup

**Snooze backoff:** Level 1: 24h, Level 2: 48h, Level 3+: 7 days. New version resets level.

**Post-upgrade:** Writes `~/.gstack/just-upgraded-from` marker, clears cache, shows changelog entries.

**Technical debt:**
- Vendored = no git history (depth-1 clone loses all)
- sed -i '' portability (macOS syntax in gstack-config)
- Snooze recorded during upgrade but immediately cleared (intentional?)

---

## Cross-Cutting Architecture

### Template System

All SKILL.md files are auto-generated from `.tmpl` templates via `bun run gen-skill-docs`. Template variables:

| Directive | Resolver | Purpose |
|-----------|----------|---------|
| `{{PREAMBLE}}` | `preamble.ts` | Tiered intro (T1-T4), upgrade check, lake intro, telemetry |
| `{{BROWSE_SETUP}}` | `browse.ts` | Headless browser setup |
| `{{SLUG_EVAL}}` | `utility.ts` | Project slug detection |
| `{{CODEX_SECOND_OPINION}}` | `constants.ts` | Codex integration |
| `{{DESIGN_SKETCH}}` | `design.ts` | Wireframe generation |
| `{{SPEC_REVIEW_LOOP}}` | `review.ts` | Subagent adversarial review |
| `{{TEST_BOOTSTRAP}}` | `testing.ts` | Test framework bootstrap |

**Preamble tier system:**

| Tier | Skills | Sections |
|------|--------|----------|
| T1 | browse, setup-cookies, benchmark | core + upgrade + lake + telemetry + contributor + completion |
| T2 | investigate, cso, retro, doc-release, setup-deploy, canary | T1 + ask format + completeness |
| T3 | autoplan, codex, design-consult, office-hours, ceo/design/eng-review | T2 + repo mode + search before building |
| T4 | ship, review, qa, qa-only, design-review, land-deploy | T3 + test failure triage |

### Shared Infrastructure

**Review dashboard:** All review skills use the same dashboard format showing runs count, last run, status (CLEAR/issues_open), whether required. Eng Review is the ONLY required gate for shipping.

**Telemetry system:** All skills write to `~/.gstack/analytics/`:
- `skill-usage.jsonl` -- skill name, timestamp, repo
- `eureka.jsonl` -- first-principles insights
- `spec-review.jsonl` -- spec review metrics
- `gstack-telemetry-log` -- binary for logging events

**State directory pattern:** `~/.gstack/` as canonical state location:
- `freeze-dir.txt` -- freeze boundary
- `config.yaml` -- user preferences
- `last-update-check` -- upgrade cache
- `just-upgraded-from` -- post-upgrade marker
- `skill-usage.jsonl` -- analytics

### Skill Chaining

```
office-hours -> design doc -> plan-ceo-review -> CEO plan -> plan-eng-review -> test plan -> /qa
                                           |
/ship -> /review (pre-landing) -> /qa-only (plan verification) -> /document-release -> /land-and-deploy -> /canary
```

### Design Philosophy Patterns

1. **Human-in-the-loop for irreversible actions:** Merge (irreversible) has pre-merge gate, VERSION bump always asks, CHANGELOG never clobbered
2. **Baked-in lessons from incidents:** CHANGELOG clobbering rule came from "a real incident"
3. **Auto-detect + persist:** Deploy platform, base branch, review staleness all auto-detected and cached
4. **Escalation paths always available:** "This is too hard for me" always acceptable, 3-attempt limit before escalation
5. **Evidence-based everything:** Screenshots for canary, diffs for document-release, consensus tables for dual voices

### Shared Binaries

| Binary | Purpose |
|--------|---------|
| `gstack-review-log` | Persist review results to `~/.gstack/projects/{slug}/{branch}-reviews.jsonl` |
| `gstack-review-read` | Read review log for dashboard display |
| `gstack-diff-scope` | Categorize diff by file type (frontend/backend/prompts/tests/docs/config) |
| `gstack-slug` | Project slug detection |
| `gstack-config` | YAML config reader/writer for `~/.gstack/config.yaml` |
| `gstack-update-check` | Version checker with caching, snooze, telemetry |

---

## Key Technical Debt Summary

### Critical (affects core workflows)

1. **Test Framework Bootstrap duplication** -- nearly identical code in `/qa` (Phase 2) and `/ship` (Step 2.5)
2. **Greptile suppression history format** -- `/review` and `/ship` both write but format doesn't include exact grep patterns
3. **Coverage diagram is ASCII** -- not programmatically parseable, can't be consumed by downstream skills
4. **Template expansion coupling** -- `{{DIRECTIVE}}` placeholders mean gen-skill-docs.ts bugs affect multiple skills simultaneously

### Moderate (affects reliability)

1. **No persistent investigation state** in `/investigate` (between sessions)
2. **Browse binary path resolution** -- version mismatch check exists but relies on path heuristics
3. **Codex/Agent tool as optional dependencies** -- degrade gracefully but lose significant capability
4. **Single page load sample** in `/benchmark` (no warm-up, no median/percentile)

### Minor (affects DX)

1. **Maxdepth 2 for doc discovery** in `/document-release` (misses deeply nested)
2. **TODOS.md parsing fragile** in `/retro` (relies on `## Completed` section naming)
3. **vendored = no git history** for gstack-upgrade (each vendored upgrade is fresh download)
4. **sed -i '' portability** in gstack-config (macOS-specific syntax)

---

## Clever Solutions

1. **Review staleness check** (`/land-and-deploy`): compares stored review commit against HEAD to detect "review done on different code than what's about to merge"

2. **Diff-scope-driven canary depth** (`/land-and-deploy`): skips expensive browser checks for docs-only, minimal smoke for config-only, full suite for frontend

3. **Dual voices degradation matrix** (`/autoplan`): both fail / one fail / both succeed each handled differently with appropriate warnings

4. **WTF-likelihood self-regulation** (`/qa`): formalizes the intuition that too many fixes in a row usually means making things worse

5. **Snapshot ref system** (`/browse`): live Playwright locators that re-resolve on every call, preventing stale DOM false failures

6. **CHANGELOG never-clobber rule** (`/document-release`): came from "a real incident" where an agent replaced existing entries

7. **Midnight-aligned windows** (`/retro`): prevents timezone drift over week-long retrospectives

8. **Trailing slash freeze boundary** (`/freeze`): `/src/` only matches `/src/`, not `/src-old`, preventing prefix-matching bypass

9. **Skill file loading from disk** (`/autoplan`): reads actual skill files, so updates to plan-ceo-review automatically incorporated

10. **Restore point capture** (`/autoplan`): backs up plan file before doing anything, enables clean resume after interruption

---

## Notable Technical Decisions

1. **SKILL.md as source of truth**: the skill definition IS the SKILL.md file. Templates generate SKILL.md files. Human-readable = machine-executable.

2. **Bash for tooling**: all helper scripts are Bash, not TypeScript. Minimal dependencies but less discoverable as coherent system.

3. **Review log at project level**: stored in `~/.gstack/projects/{slug}/`, not in repo. Enables cross-repo analysis in `/retro global` but ties history to developer's machine.

4. **Non-interactive /ship**: explicitly designed to run without asking. Shipping should be pipeline, not conversation.

5. **Fail-open for safety hooks**: if can't parse JSON input or state file missing, allow operation. Prevents blocking legitimate work but creates small window for destructive commands.

6. **PreToolUse hook composition**: Guard composes sibling skill hooks rather than reimplementing. Elegant but fragile (hard dependency on sibling installation paths).

7. **Zero-noise security output**: CSO's 8/10 confidence gate means daily mode may produce zero findings. This is intentional -- better zero findings than false positives.
