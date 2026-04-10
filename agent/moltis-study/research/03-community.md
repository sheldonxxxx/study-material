# Community Health Analysis: Moltis

## Repository Overview

| Attribute | Value |
|-----------|-------|
| **Repository** | moltis-org/moltis |
| **Language** | Rust (primary), JavaScript/TypeScript (web UI) |
| **License** | MIT (Copyright 2025 Fabien Penso) |
| **Repository Age** | Active development since ~January 2025 |
| **Activity (2025)** | 2,110 commits since January 2025 |

## Contribution Patterns

### Top Contributors

| Rank | Contributor | Commits | Percentage |
|------|-------------|---------|------------|
| 1 | Fabien Penso | 1,987 | 94.2% |
| 2 | Claude (AI agent) | 49 | 2.3% |
| 3 | github-actions[bot] | 31 | 1.5% |
| 4 | vanduc2514 | 7 | 0.3% |
| 5 | Artem | 5 | 0.2% |
| 6 | dependabot[bot] | 4 | 0.2% |
| 7 | codspeed-hq[bot] | 3 | 0.1% |
| 8 | thomas7725353 | 2 | 0.1% |

**Observation**: The repository exhibits a strongly centralized contribution pattern with Fabien Penso (the project founder/owner) accounting for over 94% of commits. External contributions are minimal, though there is evidence of some community engagement through PRs from external contributors (thomas7725353, vanduc2514).

### Merge Activity

- **Merged PRs identified**: 118 merge commits in history
- Recent external PRs merged: #484, #58, #433, #479, #480, #481, #465, #478

## GitHub Infrastructure

### Issue Templates (5 templates)

Located in `.github/ISSUE_TEMPLATE/`:

| Template | Purpose |
|----------|---------|
| `bug_report.yml` | Bug reports with expected/actual behavior, reproduction steps |
| `feature_request.yml` | Feature requests with motivation and alternatives |
| `documentation.yml` | Docs improvements and additions |
| `model_behavior.yml` | Model-specific behavior issues |
| `config.yml` | Configuration-related issues |

**Blank issues disabled** - requires contributors to use templates.

### GitHub Actions Workflows

| Workflow | Purpose |
|----------|---------|
| `ci.yml` | Main CI pipeline (16KB - comprehensive) |
| `release.yml` | Release automation (55KB - very extensive) |
| `homebrew.yml` | Homebrew package updates |
| `codspeed.yml` | Benchmarking via CodSpeed |
| `docs.yml` | Documentation deployment |
| `e2e.yml` | End-to-end testing |
| `zizmor.yml` | Workflow security linting |

### External Integrations

- **Codspeed**: Performance benchmarking integration
- **Dependabot**: Automated dependency updates
- **Homebrew**: Official Homebrew formula maintenance
- **Discord**: Community link provided in issue template config

## Release Cadence

### Versioning Strategy

- **Pattern**: Date-based versioning (`YYYYMMDD.NN`)
- **Current series**: v0.10.x (March 2026)
- **Cargo.toml**: Static version `0.1.0` with real version injected via `MOLTIS_VERSION` env var at build time

### Recent Release Activity

| Tag | Date | Notes |
|-----|------|-------|
| v0.10.18 | 2026-03-25 | Latest |
| v0.10.17 | 2026-03-05 | |
| v0.10.x series | March 2026 | 11 releases in ~3 weeks |
| v0.9.0 | 2026-02-17 | |
| v0.8.x series | Feb 2026 | Active development |

**Release cadence**: Extremely active - multiple releases per week during active development periods.

## Community Documentation

### CLAUDE.md (Agent Guide)

Comprehensive engineering guide for AI agents with:
- Rust idioms and patterns
- Testing requirements and E2E testing with Playwright
- Security practices (SSRF protection, secrets handling)
- Database migration patterns
- Release workflow documentation
- Build and validation commands

**Quality**: Exceptional - 405 lines of detailed guidance enabling AI agent collaboration.

### CONTRIBUTING.md

Clear contribution guidelines including:
- Development setup prerequisites
- Conventional commit requirements
- Validation commands (`just format-check`, `just test`, etc.)
- PR checklist (tests, formatting, session sharing)
- Security issue handling via separate SECURITY.md

### SECURITY.md

Exists with vulnerability disclosure policy.

## Community Health Indicators

### Strengths

1. **Excellent documentation for AI agents**: CLAUDE.md enables autonomous AI contribution
2. **Comprehensive CI/CD**: 6 workflows covering testing, benchmarking, releases
3. **Template-driven issue system**: Reduces low-quality reports
4. **Active release cycle**: Demonstrates active maintenance
5. **Conventional commits enforced**: Clean git history
6. **Security-conscious**: SECURITY.md, zizmor workflow, SSRF protection documented
7. **Multi-platform support**: macOS, Linux, Docker, Homebrew

### Concerns

1. **Centralized contribution model**: 94%+ commits from single maintainer
2. **Limited external contributor engagement**: Few non-bot external contributions
3. **No CODEOWNERS file**: Unclear code ownership
4. **No public ROADMAP.md visible**: Strategic direction not publicly documented
5. **Issue templates may be too complex**: Detailed templates could discourage casual contributions

### Opportunities

1. **AI agent contribution pipeline**: 49 commits from "Claude" shows successful AI-assisted development
2. **Growing release cadence**: Accelerating development suggests increasing user interest
3. **Homebrew integration**: Official package manager support
4. **Discord community**: Direct user engagement channel exists

## Recommendations

1. **Attract more reviewers**: CODEOWNERS and documented review process could encourage PRs
2. **Public roadmap**: More visible planning could attract contributors interested in specific features
3. **Good first issues**: Consider adding labeled issues for new contributors
4. **Smaller scoped PRs**: Encouraging smaller changes could increase external contribution rate

## Conclusion

Moltis demonstrates **healthy but nascent community development**. The project is well-structured with excellent documentation that actually enables AI agent collaboration. The single-maintainer model is sustainable given the comprehensive automation and clear patterns, but long-term growth would benefit from more distributed ownership. The active release cadence and growing tag frequency suggest a project in rapid development with an engaged maintainer.
