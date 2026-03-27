# gstack Feature Index

**Repository:** /Users/sheldon/Documents/claw/reference/gstack
**Generated:** 2026-03-27
**Source:** README.md + directory structure analysis

---

## Overview

gstack is an AI-augmented development workflow system that transforms Claude Code into a virtual engineering team. It provides 28 specialized skills organized around the sprint lifecycle: Think, Plan, Build, Review, Test, Ship, Reflect.

---

## Priority Tier 1: Core Workflow Skills (5-7 features)

These are the primary skills that drive the sprint workflow.

### 1. Office Hours
**Description:** YC-style startup diagnostic session. Six forcing questions that reframe product thinking before writing code. Pushes back on framing, challenges premises, generates implementation alternatives.
**Key Files:** `office-hours/SKILL.md`
**Command:** `/office-hours`

---

### 2. Plan CEO Review
**Description:** CEO/founder perspective review. Rethinks the problem to find the 10-star product hiding inside the request. Four modes: Expansion, Selective Expansion, Hold Scope, Reduction.
**Key Files:** `plan-ceo-review/SKILL.md`
**Command:** `/plan-ceo-review`

---

### 3. Plan Engineering Review
**Description:** Engineering manager perspective. Locks architecture, data flow, edge cases, and tests. Forces hidden assumptions into the open with ASCII diagrams and state machines.
**Key Files:** `plan-eng-review/SKILL.md`
**Command:** `/plan-eng-review`

---

### 4. Review (PR Review)
**Description:** Staff engineer PR review. Finds bugs that pass CI but blow up in production. Auto-fixes obvious issues, flags completeness gaps.
**Key Files:** `review/SKILL.md`
**Command:** `/review`

---

### 5. QA (Quality Assurance)
**Description:** QA Lead that tests the app, finds bugs, fixes them with atomic commits, and auto-generates regression tests for every fix.
**Key Files:** `qa/SKILL.md`
**Command:** `/qa`

---

### 6. Ship (Release Engineer)
**Description:** Release engineer workflow. Syncs main, runs tests, audits coverage, pushes, and opens PR. Bootstraps test frameworks if none exist.
**Key Files:** `ship/SKILL.md`
**Command:** `/ship`

---

### 7. Browse (Headless Browser)
**Description:** Real Chromium browser for QA. ~100ms per command with persistent state (cookies, tabs, localStorage). Long-lived daemon model with sub-second latency.
**Key Files:** `browse/src/` (CLI + server), `browse/SKILL.md`
**Commands:** `/browse`, `$B <command>`

---

## Priority Tier 2: Specialized Skills (5-8 features)

Supporting skills for specific aspects of the workflow.

### 8. Plan Design Review
**Description:** Senior designer audit. Rates each design dimension 0-10, explains what a 10 looks like, then edits the plan to get there. Includes AI slop detection.
**Key Files:** `plan-design-review/SKILL.md`
**Command:** `/plan-design-review`

---

### 9. Design Consultation
**Description:** Design partner that builds a complete design system from scratch. Researches the landscape, proposes creative risks, generates realistic product mockups.
**Key Files:** `design-consultation/SKILL.md`
**Command:** `/design-consultation`

---

### 10. Design Review
**Description:** Designer who codes. Same audit as plan-design-review, then fixes what it finds. Atomic commits with before/after screenshots.
**Key Files:** `design-review/SKILL.md`
**Command:** `/design-review`

---

### 11. QA Only
**Description:** QA reporter. Same methodology as /qa but report only without code changes. Pure bug report generation.
**Key Files:** `qa-only/SKILL.md`
**Command:** `/qa-only`

---

### 12. CSO (Chief Security Officer)
**Description:** Security audit using OWASP Top 10 + STRIDE threat model. Zero-noise output with 17 false positive exclusions, 8/10+ confidence gate, independent finding verification.
**Key Files:** `cso/SKILL.md`
**Command:** `/cso`

---

### 13. Investigate (Debugger)
**Description:** Systematic root-cause debugging. Iron Law: no fixes without investigation. Traces data flow, tests hypotheses, stops after 3 failed fixes.
**Key Files:** `investigate/SKILL.md`
**Command:** `/investigate`

---

### 14. Canary (Post-Deploy Monitoring)
**Description:** SRE post-deploy monitoring loop. Watches for console errors, performance regressions, and page failures.
**Key Files:** `canary/SKILL.md`
**Command:** `/canary`

---

### 15. Benchmark (Performance Engineering)
**Description:** Performance regression detection. Baselines page load times, Core Web Vitals, and resource sizes. Compares before/after on every PR.
**Key Files:** `benchmark/SKILL.md`
**Command:** `/benchmark`

---

## Priority Tier 3: Infrastructure & Power Tools (5-7 features)

### 16. Land and Deploy
**Description:** Release engineer single command. Merges PR, waits for CI and deploy, verifies production health. One command from "approved" to "verified in production."
**Key Files:** `land-and-deploy/SKILL.md`
**Command:** `/land-and-deploy`

---

### 17. Autoplan
**Description:** Automated review pipeline. Runs CEO, design, and eng review automatically with encoded decision principles. Surfaces only taste decisions for approval.
**Key Files:** `autoplan/SKILL.md`
**Command:** `/autoplan`

---

### 18. Document Release
**Description:** Technical writer skill. Updates all project docs to match shipped changes. Catches stale READMEs automatically.
**Key Files:** `document-release/SKILL.md`
**Command:** `/document-release`

---

### 19. Retro (Retrospective)
**Description:** Team-aware weekly retro. Per-person breakdowns, shipping streaks, test health trends, growth opportunities. Supports `/retro global` across all projects.
**Key Files:** `retro/SKILL.md`
**Command:** `/retro`

---

### 20. Setup Browser Cookies
**Description:** Session manager that imports cookies from real browser (Chrome, Arc, Brave, Edge) into headless session for authenticated page testing.
**Key Files:** `setup-browser-cookies/SKILL.md`
**Command:** `/setup-browser-cookies`

---

### 21. Setup Deploy
**Description:** One-time setup for /land-and-deploy. Detects platform, production URL, and deploy commands automatically.
**Key Files:** `setup-deploy/SKILL.md`
**Command:** `/setup-deploy`

---

### 22. Codex (Second Opinion)
**Description:** Independent code review from OpenAI Codex CLI. Three modes: review gate, adversarial challenge, open consultation. Cross-model analysis when both /review and /codex have run.
**Key Files:** `codex/SKILL.md`
**Command:** `/codex`

---

## Priority Tier 4: Safety & Utility Tools (4-5 features)

### 23. Careful (Safety Guardrails)
**Description:** Warns before destructive commands (rm -rf, DROP TABLE, force-push). Activated by saying "be careful." Override any warning.
**Key Files:** `careful/SKILL.md`
**Command:** `/careful`

---

### 24. Freeze (Edit Lock)
**Description:** Restricts file edits to one directory. Prevents accidental changes outside scope while debugging.
**Key Files:** `freeze/SKILL.md`
**Command:** `/freeze`

---

### 25. Guard (Full Safety)
**Description:** Combines /careful and /freeze for maximum safety during production work.
**Key Files:** `guard/SKILL.md`
**Command:** `/guard`

---

### 26. Unfreeze (Unlock)
**Description:** Removes the /freeze boundary to restore full editing access.
**Key Files:** `unfreeze/SKILL.md`
**Command:** `/unfreeze`

---

### 27. Gstack Upgrade
**Description:** Self-updater that upgrades gstack to latest version. Detects global vs vendored install, syncs both, shows changelog.
**Key Files:** `gstack-upgrade/SKILL.md`
**Command:** `/gstack-upgrade`

---

## Supporting Infrastructure

### Headless Browser Daemon
**Description:** Long-lived Chromium process with persistent state. First call starts (~3s), subsequent calls ~100-200ms. Supports cookies, tabs, localStorage across commands.
**Key Files:** `browse/src/` (TypeScript source), `browse/dist/` (compiled binary)

### Telemetry System
**Description:** Opt-in usage tracking stored in Supabase. Default off, asks on first run. Only skill name, duration, success/fail, version, OS sent. Local analytics via `gstack-analytics`.
**Key Files:** `lib/analytics.ts`, `scripts/gstack-telemetry-*`

### Eval Infrastructure
**Description:** LLM-judge quality evals and E2E testing framework. Two-tier system: gate (CI blocking) and periodic (weekly cron). Diff-based test selection.
**Key Files:** `test/` (skill-llm-eval.test.ts, skill-e2e-*.test.ts), `lib/eval-*.ts`

### Skill Documentation Generator
**Description:** Template-to-SKILL.md generator. Edit .tmpl files, run `bun run gen:skill-docs` to regenerate all SKILL.md files.
**Key Files:** `scripts/gen-skill-docs.ts`, `SKILL.md.tmpl`

---

## Directory Structure Reference

```
gstack/
├── office-hours/         # YC Office Hours skill
├── plan-ceo-review/     # CEO/Founder review skill
├── plan-eng-review/     # Engineering review skill
├── plan-design-review/  # Design audit skill
├── design-consultation/ # Design system builder
├── design-review/       # Design audit + fix skill
├── review/              # PR review skill
├── investigate/         # Debugger skill
├── qa/                  # QA Lead skill
├── qa-only/             # QA Reporter skill
├── cso/                 # Security audit skill
├── ship/                # Release engineer skill
├── land-and-deploy/     # Deploy verification skill
├── canary/              # Post-deploy monitoring
├── benchmark/           # Performance regression
├── document-release/    # Documentation updater
├── retro/               # Weekly retro
├── browse/              # Headless browser (Playwright)
├── setup-browser-cookies/ # Cookie importer
├── setup-deploy/        # Deploy configurator
├── autoplan/            # Auto-review pipeline
├── codex/               # Codex second opinion
├── careful/             # Safety guardrails
├── freeze/              # Edit lock
├── guard/               # Full safety mode
├── unfreeze/            # Edit unlock
├── gstack-upgrade/      # Self-updater
├── lib/                  # Core utilities (analytics, eval helpers)
├── scripts/             # Build/DX tooling
├── bin/                 # CLI utilities
├── test/                # Test suite
└── supabase/            # Telemetry schema
```
