# OpenFang Community Health Report

## Repository Overview

| Metric | Value |
|--------|-------|
| Stars | 150 |
| Open Issues | 30 |
| Description | Open-source Agent Operating System |
| Topics | agent-framework, ai-agents, llm, mcp, open-source, openclaw, operating-system, rust |
| Latest Release | v0.5.2 (2026-03-26) - "12 Bug Fixes" |
| License | MIT + Apache 2.0 (dual) |

## Release Cadence

The project shows **very active development** with rapid releases:

- v0.1.0 - 2026-02-24 (initial release)
- v0.5.0 - recent (feature release)
- v0.5.1 - recent (patch)
- v0.5.2 - 2026-03-26 (latest)

Approximately **5 releases in ~1 month**, indicating an active launch phase with frequent iteration.

## Git Activity

**Recent commits (2025-01-01 to present):** 176 commits

**Last commit dates:**
- 2026-03-26 (most recent)
- 2026-03-20
- 2026-03-19
- 2026-03-18
- 2026-03-16

The project shows **daily or near-daily commits** with recent activity.

## Contributors

**Total unique contributors:** 10

| Contributor | Commits | Email |
|-------------|---------|-------|
| jaberjaber23 | 132 | jaberib647@gmail.com |
| Jaber Jaber | 9 | 103749727+jaberjaber23@users.noreply.github.com |
| dependabot[bot] | 8 | 49699333+dependabot[bot]@users.noreply.github.com |
| Mark B | 3 | mark@vpost.net |
| Liu | 2 | lc-soft@live.cn |
| Irwin | 2 | irwingem8@hotmail.com |
| Frank | 2 | 97429702+tsubasakong@users.noreply.github.com |
| Evan Hu | 2 | suzukaze.haduki@gmail.com |
| tuzkier | 1 | (GitHub auth) |
| reaster | 1 | (GitHub auth) |

**Concentration:** Single dominant contributor (jaberjaber23) with ~75% of all commits. This is typical for a project in early development but represents a **bus factor risk**.

## Governance Files

### Present

| File | Status |
|------|--------|
| CONTRIBUTING.md | Yes - 372 lines, comprehensive |
| Issue Templates | Yes (Bug Report, Feature Request) |
| PR Template | Yes |
| FUNDING.yml | Yes |
| dependabot.yml | Yes |
| GitHub Workflows | Yes |
| Code of Conduct | Contributor Covenant v2.1 |

### Missing

| File | Status |
|------|--------|
| CODEOWNERS | No |
| MAINTAINERS | No |
| AUTHORS | No |

## Contributing Guidelines Quality

The CONTRIBUTING.md is **comprehensive and well-structured**:

- Development environment setup (Rust 1.75+, Git, Python)
- Build and testing instructions (1,744+ tests)
- Code style guidelines (rustfmt, clippy, doc comments)
- Architecture overview (14-crate workspace)
- Step-by-step guides for adding:
  - New agent templates
  - New channel adapters
  - New tools
- PR process with review requirements
- Commit message conventions

## Dependency Management

- **dependabot** is active (8 automated PRs for cargo/zip, roxmltree, docker actions)
- Indicates professional maintenance of dependency hygiene

## Community Health Summary

| Dimension | Assessment |
|-----------|------------|
| Activity Level | Very High (daily commits, rapid releases) |
| Contributor Diversity | Low (single dominant contributor) |
| Documentation | Excellent (comprehensive CONTRIBUTING.md) |
| Templates | Good (issue/PR templates present) |
| Governance | Minimal (no CODEOWNERS/MAINTAINERS/AUTHORS) |
| Open Source Maturity | Early-stage (launch phase, v0.5.x) |

## Key Observations

1. **Rapid development pace** - Multiple releases per week suggests active development and frequent shipping
2. **Single maintainer risk** - 75% of commits from one contributor creates bus factor concern
3. **Professional tooling** - dependabot, CI workflows, comprehensive CONTRIBUTING.md indicate production-grade project
4. **Early-stage community** - Low contributor count but active external contributions (from git history)
5. **Responsive to issues** - Recent commit "fix: resolve 6 more bugs (#845, #844, #823, #767, #802, #816)" suggests active bug triaging

## Recommendations for Project Health

1. Add CODEOWNERS to distribute review responsibility
2. Consider MAINTAINERS file to identify secondary maintainers
3. Build more diverse contributor base as project matures
