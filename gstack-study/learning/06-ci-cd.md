# CI/CD

## Overview

gstack CI uses GitHub Actions with a **pre-baked Docker image** strategy. The Docker image contains the full toolchain (Bun, Playwright, Chromium), so CI runners don't need to install anything — they just pull the image.

**Key design decisions:**
- Docker image cached by `package.json` hash — only rebuilds when dependencies change
- Evals run on Ubicloud standard runners (not GitHub-hosted), reducing cost
- 12 parallel eval suites per PR, each in its own Docker container
- Diff-based test selection — only tests affected by the change run

## Workflows

### 1. skill-docs.yml — Docs Freshness Check

**Trigger:** Every push and PR.

**Purpose:** Ensures SKILL.md files are regenerated after any template change.

```yaml
on: [push, pull_request]

jobs:
  check-freshness:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: bun install
      - run: bun run gen:skill-docs
      - run: git diff --exit-code  # fails if SKILL.md is stale
      - run: bun run gen:skill-docs --host codex
      - run: git diff --exit-code -- .agents/  # fails if Codex SKILL.md is stale
```

If generated files differ from what's committed, CI fails with:
```
Generated SKILL.md files are stale. Run: bun run gen:skill-docs
```

### 2. ci-image.yml — Docker Image Build

**Trigger:**
- Weekly (Monday 6am UTC) — pick up CLI updates
- On push to main when `Dockerfile.ci` or `package.json` changes
- Manual via `workflow_dispatch`

**Purpose:** Build and push the `ghcr.io/<owner>/gstack/ci` Docker image.

```yaml
runs-on: ubicloud-standard-2
steps:
  - uses: actions/checkout@v4
  - run: cp package.json .github/docker/
  - uses: docker/login-action@v3 (ghcr.io)
  - uses: docker/build-push-action@v6
      file: .github/docker/Dockerfile.ci
      tags: |
        ghcr.io/${{ github.repository }}/ci:latest
        ghcr.io/${{ github.repository }}/ci:${{ github.sha }}
```

### 3. evals.yml — E2E Eval Suite

**Trigger:** Every PR to main + manual dispatch.

**Concurrency:** One eval run per PR branch; cancels in-progress if new commits arrive.

**Architecture:**
```
build-image (ubicloud-standard-2)
    ↓ (on PR)
evals (12 parallel suites, each in Docker container)
    ↓ (on complete)
report (posts PR comment with results)
```

**Eval suites:**

| Suite | File | Runner | Purpose |
|-------|------|--------|---------|
| llm-judge | `test/skill-llm-eval.test.ts` | ubicloud-standard-2 | LLM-as-judge scoring |
| e2e-browse | `test/skill-e2e-bws.test.ts` | ubicloud-standard-8 | Browser skill E2E |
| e2e-plan | `test/skill-e2e-plan.test.ts` | ubicloud-standard-2 | Planning skills E2E |
| e2e-deploy | `test/skill-e2e-deploy.test.ts` | ubicloud-standard-2 | Deploy skills E2E |
| e2e-design | `test/skill-e2e-design.test.ts` | ubicloud-standard-2 | Design skills E2E |
| e2e-qa-bugs | `test/skill-e2e-qa-bugs.test.ts` | ubicloud-standard-2 | QA bug-finding E2E |
| e2e-qa-workflow | `test/skill-e2e-qa-workflow.test.ts` | ubicloud-standard-2 | QA workflow E2E |
| e2e-review | `test/skill-e2e-review.test.ts` | ubicloud-standard-2 | Review skills E2E |
| e2e-workflow | `test/skill-e2e-workflow.test.ts` | ubicloud-standard-2 | Full workflow E2E |
| e2e-routing | `test/skill-routing-e2e.test.ts` | ubicloud-standard-2 | Skill routing E2E |
| e2e-codex | `test/codex-e2e.test.ts` | ubicloud-standard-2 | Codex integration E2E |
| e2e-gemini | `test/gemini-e2e.test.ts` | ubicloud-standard-2 | Gemini integration E2E |

**Docker image caching strategy:**
- Image tag = `hash(Dockerfile.ci, package.json)`
- If image exists in registry, skip build
- If not, build and push
- Runner restores `node_modules` via symlink from `/opt/node_modules_cache` if `.package.json` matches

**E2E execution environment:**
```bash
ANTHROPIC_API_KEY: secrets.ANTHROPIC_API_KEY
OPENAI_API_KEY: secrets.OPENAI_API_KEY
GEMINI_API_KEY: secrets.GEMINI_API_KEY
EVALS_CONCURRENCY: "40"
PLAYWRIGHT_BROWSERS_PATH: /opt/playwright-browsers
EVALS=1 bun test --retry 2 --concurrent --max-concurrency 40 <test-file>
```

**PR comment reporting:**
After all suites complete, the `report` job parses eval JSON results and posts a PR comment:
```
## E2E Evals: ✅ PASS

${PASSED}/${TOTAL} tests passed | $${COST} total cost | 12 parallel runners

| Suite | Result | Status | Cost |
|-------|--------|--------|------|
| e2e-browse | 12/12 | ✅ | $0.32 |
| ...
```

### 4. evals-periodic.yml — Weekly Full Eval

**Trigger:** Weekly cron (not on every PR).

Runs ALL tests regardless of diff (`EVALS_ALL=1`). Classified as `periodic` tier — gates on main only.

## Docker Image

**Dockerfile.ci location:** `.github/docker/Dockerfile.ci`

The image includes:
- Bun runtime
- Playwright with Chromium
- All dependencies from `package.json`

Cached at `/opt/node_modules_cache` for fast dependency restoration.

## Test Tiers

| Tier | Command | Cost | What it tests |
|------|---------|------|---------------|
| 1 — Static | `bun test` | Free | Command validation, snapshot flags, SKILL.md correctness, observability unit tests |
| 2 — E2E | `bun run test:e2e` | ~$3.85 | Full skill execution via `claude -p` subprocess |
| 3 — LLM eval | `bun run test:evals` | ~$0.15 standalone | LLM-as-judge scoring of generated SKILL.md docs |
| 2+3 | `bun run test:evals` | ~$4 combined | E2E + LLM-as-judge |

**Diff-based test selection:** `test:evals` auto-selects tests based on `git diff` against base branch. Each test declares file dependencies in `test/helpers/touchfiles.ts`. Use `EVALS_ALL=1` to force all tests.

**Two-tier system:**
- `gate` tier — runs on every PR (safety guardrails, deterministic tests)
- `periodic` tier — weekly cron or manual (quality benchmarks, Opus model tests, non-deterministic)

## Eval Observability

E2E tests produce artifacts in `~/.gstack-dev/`:

| Artifact | Path | Purpose |
|----------|------|---------|
| Heartbeat | `e2e-live.json` | Current test status (updated per tool call) |
| Partial results | `evals/_partial-e2e.json` | Completed tests (survives kills) |
| Progress log | `e2e-runs/{runId}/progress.log` | Append-only text log |
| NDJSON transcripts | `e2e-runs/{runId}/{test}.ndjson` | Raw `claude -p` output per test |
| Failure JSON | `e2e-runs/{runId}/{test}-failure.json` | Diagnostic data on failure |

**Live dashboard:** `bun run eval:watch` shows real-time progress.

**Comparison tools:**
```bash
bun run eval:list      # list all eval runs
bun run eval:compare   # compare two runs — per-test deltas + Takeaway commentary
bun run eval:summary   # aggregate stats + per-test efficiency averages
```

## Local Development CI Simulation

```bash
bun test              # Tier 1 only (free, <2s)
bun run test:evals    # Tier 2+3 combined (~$4, diff-based)
```

E2E tests stream progress in real-time: `[Ns] turn T tool #C: Name(...)`.
