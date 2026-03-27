# CI/CD Pipeline Analysis

## Overview

GSD uses **GitHub Actions** with a focused pipeline that prioritizes security scanning, cross-platform testing, and rapid feedback.

## Workflows

### 1. Test Workflow (`.github/workflows/test.yml`)

**Trigger:**
- Push to `main` branch
- Pull requests to `main` branch
- Manual dispatch (`workflow_dispatch`)

**Matrix Strategy:**

| OS | Node Version | Purpose |
|----|--------------|---------|
| `ubuntu-latest` | 22, 24 | Primary testing (fail-fast) |
| `macos-latest` | 22 | Platform compatibility |
| `windows-latest` | 22 | Smoke test (slowest runner) |

**Concurrency Control:**
```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true
```

**Steps:**
1. Checkout with full history (`fetch-depth: 0`)
2. Setup Node.js with npm caching
3. `npm ci` — clean install
4. `npm run test:coverage` — full test suite with coverage

**Coverage Requirement:** 70% line coverage on `get-shit-done/bin/lib/*.cjs`

### 2. Security Scan Workflow (`.github/workflows/security-scan.yml`)

**Trigger:** Pull requests to `main` branch only

**Timeout:** 5 minutes

**Scans Performed:**

| Scan | Script | Purpose |
|------|--------|---------|
| Prompt injection | `prompt-injection-scan.sh` | Detects embedded injection vectors in prompts |
| Base64 obfuscation | `base64-scan.sh` | Finds obfuscated code/commands |
| Secret scanning | `secret-scan.sh` | Prevents credentials in commits |
| Planning directory | Git diff check | Ensures `.planning/` not committed |

**Why only PR?** Security scans on every push would slow down main branch development unnecessarily.

### 3. Auto-Label Issues (`.github/workflows/auto-label-issues.yml`)

Automatic issue labeling based on templates.

## Security Scanning Details

### Prompt Injection Scanner

```bash
chmod +x scripts/prompt-injection-scan.sh
scripts/prompt-injection-scan.sh --diff "origin/$BASE_REF"
```

Detects common prompt injection patterns in agent/workflow/command files.

### Base64 Scanner

```bash
scripts/base64-scan.sh --diff "origin/$BASE_REF"
```

Flags potentially obfuscated code that could hide malicious content.

### Secret Scanner

```bash
scripts/secret-scan.sh --diff "origin/$BASE_REF"
```

Prevents API keys, tokens, and credentials from entering the repository.

### Planning Directory Guard

```bash
PLANNING_FILES=$(git diff --name-only --diff-filter=ACMR "origin/$BASE_REF"...HEAD | grep '^\.planning/' || true)
```

Runtime data (`.planning/`) must never be committed — it contains project-specific state that shouldn't pollute the GSD repo itself.

## CI Configuration

### Node.js Setup
```yaml
uses: actions/setup-node@53b83947a5a98c8d113130e565377fae1a50d02f  # v6.3.0
with:
  node-version: ${{ matrix.node-version }}
  cache: 'npm'
```

### Fail-Fast Strategy

```yaml
strategy:
  fail-fast: true
  matrix:
    os: [ubuntu-latest]
    node-version: [22, 24]
```

Ubuntu runs both Node versions (full matrix). macOS and Windows get only Node 22 as a smoke test.

## PR Requirements

From `CONTRIBUTING.md`:

> CI must pass — all matrix jobs (Ubuntu, macOS, Windows x Node 22, 24) must be green

PR template requires:
- [ ] All platforms tested (macOS, Windows, Linux)
- [ ] All runtimes tested (Claude Code, Gemini CLI, OpenCode, Codex, Copilot)
- [ ] Existing tests pass (`npm test`)
- [ ] No unnecessary dependencies added
- [ ] Windows backslash paths tested
- [ ] CHANGELOG.md updated for user-facing changes

## Release Pipeline

- **Version:** `package.json` `version` field (e.g., `1.29.0`)
- **Release Script:** `.release-monitor.sh` (monitoring script)
- **npm Publish:** Via `npm publish` after version bump and CHANGELOG update
- **Git Tagging:** Per milestone via `/gsd:complete-milestone`

## What GSD Does NOT Have

- No Docker containerization
- No staging/production environments
- No deployment automation beyond npm publish
- No integration with external CI providers
- No E2E testing pipeline

This is a **CLI tool package** — not a running service. CI focuses on:
1. Does it install correctly?
2. Do tests pass?
3. Is it secure?
4. Does it work across platforms?

## Coverage Metrics

```bash
npm run test:coverage
# c8 --check-coverage --lines 70 \
#    --include 'get-shit-done/bin/lib/*.cjs' \
#    --exclude 'tests/**' --all
```

70% line coverage is enforced on all core library modules. Tests themselves are excluded from coverage requirements (they are the tests, not the product).
