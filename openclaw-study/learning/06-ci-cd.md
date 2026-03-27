# CI/CD - Learning Document

**Prerequisites:**
- Review `research/02-tech-stack.md` for Docker and workflow details
- Review `research/07-code-quality.md` for pre-commit hooks and detect-secrets

---

## Overview

OpenClaw employs a comprehensive CI/CD infrastructure with GitHub Actions, multi-platform testing, Docker containerization, and automated release workflows. The project validates code across Linux, macOS, Windows, and Android simultaneously.

---

## 1. GitHub Actions Workflows

### Workflow Overview

| Workflow | Purpose |
|----------|---------|
| `ci.yml` | Main CI — lint, type check, test, coverage |
| `ci-bun.yml` | Bun runtime-specific tests |
| `codeql.yml` | CodeQL security analysis |
| `docker-release.yml` | Docker image publishing |
| `macos-release.yml` | macOS app release |
| `openclaw-npm-release.yml` | NPM package release |
| `plugin-npm-release.yml` | Plugin NPM releases |
| `stale.yml` | Stale issue/PR management |
| `labeler.yml` | PR label automation |
| `auto-response.yml` | Automated responses to issues |
| `install-smoke.yml` | Installation verification |
| `sandbox-common-smoke.yml` | Sandbox smoke tests |
| `workflow-sanity.yml` | Workflow validation |

### CI Preflight System

Dynamic matrix generation based on changed files:

```javascript
// Scripts output via scripts/ci-write-manifest-outputs.mjs
// Detects:
// - docs-only changes
// - Scope changes (Node, macOS, Android, Windows)
// - Extension changes
```

### Runners

| Runner | Platform |
|--------|----------|
| `blacksmith-16vcpu-ubuntu-2404` | Primary Linux |
| `blacksmith-32vcpu-windows-2025` | Windows (32 vCPU) |
| `macos-latest` | macOS |

### Quality Gates

All must pass before merge:
- **Lint/Format:** Oxlint, Oxfmt, SwiftLint, SwiftFormat, ShellCheck
- **Type Check:** TypeScript with `noEmitOnError`
- **Unit Tests:** Vitest with coverage thresholds
- **Security:** CodeQL + detect-secrets
- **Dependency Audit:** pnpm audit at pre-commit and CI

---

## 2. Docker Setup

### Base Images

| Image | Purpose |
|-------|---------|
| `node:24-bookworm` | Default runtime |
| `node:24-bookworm-slim` | Slim variant |
| `debian:bookworm-slim` | Sandbox base |

### Multi-Stage Dockerfile

**Stages:**

1. `ext-deps` — Extract extension package.json files
2. `build` — Install dependencies, run build
3. `runtime-assets` — Prune dev deps, strip type files
4. `base-default` / `base-slim` — Runtime base
5. `runtime` — Final runtime image

### Build Arguments

```dockerfile
OPENCLAW_EXTENSIONS=""      # Opt-in extensions
OPENCLAW_VARIANT="default"  # or "slim"
OPENCLAW_INSTALL_BROWSER="" # Install Chromium + Xvfb
OPENCLAW_INSTALL_DOCKER_CLI="" # Docker CLI for sandbox
OPENCLAW_DOCKER_APT_PACKAGES="" # Extra system packages
```

### Docker Compose

**Services:**
- `openclaw-gateway` — Gateway server (port 18789)
- `openclaw-cli` — CLI client (networked to gateway)

---

## 3. Pre-commit Hooks

### Hook Configuration

From `.pre-commit-config.yaml`:

| Hook | Tool | Purpose | Severity |
|------|------|---------|----------|
| File hygiene | pre-commit-hooks | trailing-whitespace, end-of-file, yaml checks, private key detection | error |
| Secret detection | detect-secrets | Secret scanning | error |
| Shell linting | shellcheck | Shell script validation | error |
| GitHub Actions | actionlint | Workflow validation | error |
| Actions security | zizmor | Security audit for workflows | error |
| Python linting | ruff | Python skills linting | error |
| Python tests | pytest | Python skills tests | error |
| pnpm audit | pnpm audit | Production dependency audit | error |
| TypeScript lint | oxlint | JS/TS linting | error |
| TypeScript fmt | oxfmt | Formatting | error |
| Swift lint | swiftlint | iOS/macOS linting | error |
| Swift fmt | swiftformat | iOS/macOS formatting | error |

### Pre-commit Tools Versions

| Tool | Version |
|------|---------|
| oxlint | 1.56.0 |
| oxfmt | 0.41.0 |
| oxlint-tsgolint | 0.17.1 |
| detect-secrets | 1.5.0 |
| shellcheck | (latest) |
| ruff | 0.14.1 |
| swiftlint | (latest) |
| swiftformat | (latest) |

### Hook Execution Flow

1. File hygiene checks (whitespace, EOF)
2. Secret detection (detect-secrets)
3. Shell script linting (shellcheck)
4. GitHub Actions validation (actionlint)
5. GHA security audit (zizmor)
6. Python linting/tests (ruff, pytest) — for skills/
7. Production dependency audit (pnpm audit)
8. TypeScript linting (oxlint)
9. TypeScript formatting (oxfmt)
10. Swift linting (swiftlint)
11. Swift formatting (swiftformat)

---

## 4. Release Process

### NPM Release Workflows

#### openclaw-npm-release.yml

```yaml
# Triggered on version tags
on:
  push:
    tags:
      - 'v*'
```

#### plugin-npm-release.yml

- Handles plugin package releases separately
- Each plugin can have its own release flow

### Docker Release Workflow

**docker-release.yml:**
- Builds multi-platform images (linux/amd64, linux/arm64)
- Publishes to container registry
- Supports both `default` and `slim` variants
- Extension packages included via `OPENCLAW_EXTENSIONS` build arg

### macOS Release

**macos-release.yml:**
- Builds app bundle
- Codesigns application
- Notarizes for distribution
- Creates DMG or zip archive

### Release Trigger Pattern

```yaml
on:
  push:
    tags:
      - 'v*'  # e.g., v1.2.3
```

### Version Coordination

The `openclaw-release-maintainer` skill handles:
- Release naming
- Version coordination across packages
- Changelog generation

---

## 5. Branch Protection

### CODEOWNERS

```bash
# .github/CODEOWNERS
# Security-focused paths have restricted access
```

Security-sensitive areas have limited code owners.

### PR Requirements

The comprehensive PR template enforces:

- **Security Impact** — Required section with 5 specific questions
- **Regression Test Plan** — Specifies what coverage should have caught issues
- **Human Verification** — Reviewer must personally verify, not just rely on CI
- **Root Cause Analysis** — For bugs, demands root cause and regression history
- **Review Conversations** — Bot conversation resolution required before merge

### Status Checks

Required checks before merge:
- CI pipeline passing (all platforms)
- No secrets detected
- CodeQL analysis clean
- Coverage thresholds met

---

## 6. Dependabot Setup

### Security Updates

While not explicitly detailed in researched files, the project structure supports:
- Automated security updates via GitHub Actions
- `dependabot.yml` configuration likely present
- pnpm workspace dependencies tracked

### Dependency Audit

**pnpm audit** runs at:
- Pre-commit hooks (blocks commits with high/critical vulnerabilities)
- CI pipeline

---

## 7. Multi-Platform CI

### Platform Matrix

| Platform | Status | CI Task |
|----------|--------|---------|
| Linux (Ubuntu 24.04) | Primary | Full test suite |
| macOS | Supported | Build + test |
| Windows | Tested | 32 vCPU runner |
| Android | Supported | Gradle builds |
| iOS | Supported | Swift build + test |

### Android CI Tasks

```bash
android:assemble       # Debug APK
android:bundle:release # AAB release
android:test           # Unit tests
android:test:integration # Live integration tests
```

### iOS CI Tasks

```bash
ios:build              # iOS simulator build
mac:package            # macOS app bundle
swiftlint              # Linting
swiftformat --lint     # Formatting
swift build            # Package build
swift test             # Tests
```

---

## 8. Workflow Validation

### actionlint

Validates GitHub Actions YAML syntax and common mistakes.

### zizmor

Security audit tool for GitHub Actions workflows:
- Checks for insecure workflow patterns
- Validates credential handling
- Flags potential security issues

### workflow-sanity.yml

Validates workflow files are properly configured before merging.

---

## 9. Key CI/CD Patterns

### Dynamic Matrix Generation

```javascript
// Based on changed files, determine what to run:
// - docs-only? Skip most tests
// - Node changes? Run full test suite
// - Extension changes? Run extension tests
// - Platform-specific? Run only that platform
```

### Test Distribution

- Workers adaptive: `maxWorkers: isCI ? 3 : localWorkers`
- Timeout: 120s default (180s on Windows)
- Fork pool prevents test pollution

### Build Caching

- pnpm cache in CI
- Build artifacts preserved between stages
- Docker layer caching

---

## 10. Security Automation

### detect-secrets in CI

- Baseline maintained in `.secrets.baseline`
- Prevents new secrets from entering repository
- CI blocks commits with detected secrets

### CodeQL

Automated security analysis:
- Runs on every PR
- Scans for vulnerabilities
- Generated detailed security report

### Dependency Scanning

- pnpm audit in pre-commit
- Security advisories checked
- Known vulnerabilities block builds

---

## Summary

| Aspect | Implementation |
|--------|----------------|
| CI Platform | GitHub Actions |
| Test Runners | Vitest (Node), pytest (Python), swift test (Swift) |
| Container | Docker multi-stage (Node 24 bookworm) |
| Lint/Format | Oxlint, Oxfmt, SwiftLint, SwiftFormat, ShellCheck, ruff |
| Secrets | detect-secrets (pre-commit + CI) |
| Security | CodeQL, zizmor, dependency audit |
| Release | NPM, Docker, macOS app |
| Multi-platform | Linux, macOS, Windows, Android, iOS |

---

## Key Takeaways

1. **Comprehensive CI** — Lint, type, test, coverage all enforced
2. **Multi-platform validation** — Linux, macOS, Windows, Android, iOS all tested
3. **Docker-first deployment** — Multi-stage builds, slim variants
4. **Pre-commit blocks problems** — Secrets, lint, format all enforced locally
5. **Release automation** — Tags trigger full release pipelines
6. **Security baked in** — CodeQL, detect-secrets, zizmor all automated
7. **PR template enforces quality** — Security review, root cause, human verification required

---

## References

- Workflows: `.github/workflows/ci.yml`, `.github/workflows/docker-release.yml`
- Pre-commit: `.pre-commit-config.yaml`
- Docker: `Dockerfile`, `docker-compose.yml`
- Secrets: `.secrets.baseline`, `.detect-secrets.cfg`
- Mobile CI: `apps/android/`, `apps/ios/`
