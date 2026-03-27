# gstack Community Health Report

## Repository Overview

| Metric | Value |
|--------|-------|
| **Repository** | garrytan/gstack |
| **URL** | https://github.com/garrytan/gstack |
| **Stars** | 49,272 |
| **Forks** | 6,257 |
| **Description** | "Use Garry Tan's exact Claude Code setup: 15 opinionated tools that serve as CEO, Designer, Eng Manager, Release Manager, Doc Engineer, and QA" |
| **Last Push** | 2026-03-26T14:31:53Z |

## Governance Structure

### Governance Files (MISSING)
- No CODEOWNERS file
- No MAINTAINERS file
- No AUTHORS file
- No CONTRIBUTING guidelines

### GitHub Templates (MISSING)
- No issue templates
- No PR templates

### Assessment
The repository lacks formal governance documentation. This is typical for single-maintainer or small-team projects but may limit community contribution pathways.

## GitHub Infrastructure

### .github/ Contents
```
.github/
├── actionlint.yaml        # Workflow linting config
├── docker/
│   └── Dockerfile.ci      # Pre-baked CI toolchain (Bun + Playwright)
└── workflows/
    ├── actionlint.yml     # Lint GitHub Actions
    ├── ci-image.yml       # Build Docker image
    ├── evals-periodic.yml # Weekly eval runs
    ├── evals.yml          # E2E tests on PR (12 parallel runners)
    └── skill-docs.yml     # Generate SKILL.md from templates
```

### CI/CD Assessment
Strong CI/CD infrastructure:
- Docker-based reproducible build environment
- 12 parallel Ubicloud runners for E2E tests
- Two-tier test system: gate (required) + periodic (weekly)
- Actionlint for workflow validation
- LLM-judge quality evaluation
- Diff-based test selection to reduce cost

## Versioning and Releases

### Versioning Strategy
- Semantic versioning (v0.11.x.x pattern)
- Tags created for each release
- Latest: v0.11.21.0 (2026-03-26)

### Release Cadence
Very active development with multiple releases per day:
- 2026-03-26: v0.11.21.0, v0.11.20.0
- 2026-03-25: v0.11.19.0
- 2026-03-24: v0.11.18.2, v0.11.12.1, v0.11.13.0, v0.11.16.0, v0.11.16.1, v0.11.11.0

### Release Mechanism
Uses git tags for versioning. GitHub Releases not detected via API (latestRelease: null).

## Repository Activity

### Commit Frequency (last 7 days)
| Date | Commits |
|------|---------|
| 2026-03-26 | 2 |
| 2026-03-25 | 1 |
| 2026-03-24 | 10 |
| 2026-03-23 | 9 |
| 2026-03-22 | 12 |
| 2026-03-21 | 7 |
| 2026-03-20 | 5 |

### Activity Assessment
Highly active repository with daily commits. Sustained high-velocity development over at least the past week.

## Open Issues and PRs

### Open Issues (sample)
| # | Title |
|---|-------|
| 514 | Fix Claude plan path leakage in generated Codex skills |
| 513 | /office-hours: Surface founder signal reasoning in YC recommendation |
| 511 | feat: Flutter Web support for browse snapshot |
| 510 | Context warnings trigger too early on 1M context window models |
| 509 | feat: /oracle - Product Conscience with AST-powered codebase scanner |

### Open PRs
Multiple PRs in review state visible (514, 511, 509, 506, 505).

## Top Contributors

Based on git history, garrytan is the primary contributor. The repository appears to be a single-maintainer project with external contributions via PRs.

## Community Health Summary

### Strengths
1. **High velocity** - Multiple commits and releases daily
2. **Strong CI/CD** - Comprehensive E2E testing with 12 parallel runners
3. **Large following** - 49K stars indicates significant community interest
4. **Active engagement** - Multiple open issues and PRs being worked

### Weaknesses
1. **No governance docs** - No CONTRIBUTING, CODEOWNERS, MAINTAINERS, or AUTHORS
2. **No templates** - Missing issue/PR templates
3. **Single maintainer** - Limited bus factor
4. **No GitHub Releases** - Uses tags only, no formal release notes

### Recommendations
1. Add CONTRIBUTING.md to guide external contributors
2. Create issue and PR templates for consistency
3. Consider adding CODEOWNERS for review assignment
4. Add MAINTAINERS.md if accepting additional maintainers
5. Consider GitHub Releases with changelog for user-facing releases

## Files Analyzed
- `.github/` directory structure
- `.github/workflows/*.yml` (6 workflow files)
- `.github/docker/Dockerfile.ci`
- `.github/actionlint.yaml`
- Git history (last 30 commits)
- Git tags
- GitHub API: repo metadata, contributors, issues, PRs, releases
