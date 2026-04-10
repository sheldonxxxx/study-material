# Community

## Repository Overview

| Field | Value |
|-------|-------|
| **Repository** | CortexReach/memory-lancedb-pro |
| **Package** | memory-lancedb-pro (npm) |
| **License** | MIT |
| **Author** | win4r |
| **Type** | OpenClaw Plugin (Node.js/TypeScript) |

## Activity Levels

### Commit Activity

| Period | Commit Count |
|--------|--------------|
| March 2026 | 376 |
| February 2026 | 46 |
| 2025 (full year) | 160 |
| **2026 YTD** | **422+** |

March 2026 alone accounts for nearly as many commits as all of 2025, indicating **very high recent activity** driven by OpenClaw 2026.3+ hook migration and new feature releases.

### Top Contributors

| Rank | Contributor | Commits (All Time) |
|------|-------------|-------------------|
| 1 | Ubuntu | 100 |
| 2 | 安闲静雅 | 96 |
| 3 | furedericca | 51 |
| 4 | 陈基勇 | 48 |
| 5 | Rwm Jhb | 27 |
| 6 | chenjiyong | 25 |
| 7 | win4r | 20 |
| 8 | Chao Qin | 12 |
| 9 | Hi-Jiajun | 8 |
| 10 | ssyn0813 | 7 |

**50+ unique contributors** total. Strong geographic diversity with significant contributions from Chinese-speaking developers (Ubuntu, 安闲静雅, 陈基勇, chenjiyong, Chao Qin) alongside international contributors (furedericca, Rwm Jhb, ssyn0813).

## Release Cadence

| Version | Type | Notable Changes |
|---------|------|-----------------|
| 1.1.0-beta.10 | Beta | OpenClaw 2026.3+ hook adaptation |
| 1.1.0-beta.2 | Beta | Smart Memory Beta + Access Reinforcement |
| 1.1.0-beta.1 | Beta | Initial Smart Memory Beta |
| 1.0.26 | Stable | Access Reinforcement for Time Decay |
| 1.0.22 | Stable | Storage Path Validation & Better Error Messages |
| 1.0.21 | Stable | Long Context Chunking |
| ... | ... | ... |
| 1.0.0 | Stable | Initial npm release |

- **Regular releases** -- average of every few weeks
- **Beta track active** -- v1.1.0-beta.x series for new features
- **Stable track maintained** -- v1.0.x for production use
- **Backward compatibility prioritized** -- rare breaking changes

## Internationalization

### README Translations

**Exceptional coverage -- 11 languages:**

| Language | File | Notes |
|----------|------|-------|
| English | `README.md` | Primary |
| Simplified Chinese | `README_CN.md` | 32KB |
| Traditional Chinese | `README_TW.md` | 32KB |
| Japanese | `README_JA.md` | 39KB |
| Korean | `README_KO.md` | 36KB |
| French | `README_FR.md` | 36KB |
| Spanish | `README_ES.md` | 37KB |
| German | `README_DE.md` | 35KB |
| Italian | `README_IT.md` | 35KB |
| Russian | `README_RU.md` | 46KB (largest) |
| Portuguese (Brazil) | `README_PT-BR.md` | 35KB |

### Additional Documentation Translations

- `docs/openclaw-integration-playbook.zh-CN.md` -- Chinese translation of integration guide

This level of localization is exceptional for a plugin project and indicates a global user base with active international community contributions.

## Contributing Infrastructure

### Issue Templates

**bug_report.yml:**
- Requires plugin version and OpenClaw version
- Bug description, expected behavior, steps to reproduce (all required)
- Error logs/screenshot field
- Embedding provider dropdown
- OS/platform information field

**feature_request.yml:**
- Problem/motivation section (required)
- Proposed solution section (required)
- Alternatives considered section
- Scope dropdown (Retrieval, Storage, Embedding, CLI, Configuration, Auto-capture/auto-recall, Scopes/Access Control, Migration, Documentation, Other)

**config.yml:**
- Links to Discord community (https://discord.com/invite/clawd)
- Blank issues disabled

### CI/CD Workflows

| Workflow | Purpose |
|----------|---------|
| `ci.yml` | Version sync check + npm test on push/PR |
| `auto-assign.yml` | Auto-assign issues to AliceLJY, PRs to rwmjhb |
| `claude.yml` | Claude Code review integration on mentions |
| `claude-code-review.yml` | Automated code review on PRs |

### Missing Contributing Files

- **No CONTRIBUTING.md** -- no explicit contributing guidelines document
- **No PR template** -- no pull request template detected
- **No CODEOWNERS** -- no automatic reviewer assignment by path
- **No MAINTAINERS/AUTHORS file** -- no formal maintainer listings

## Support Channels

- **Discord** -- primary support channel (https://discord.com/invite/clawd), referenced in issue template and README
- **GitHub Issues** -- bug reports and feature requests via issue templates
- **npm** -- package distribution via npm registry

## Ecosystem

Community-built tools surrounding the plugin:

- **CortexReach/toolbox** -- setup scripts repository
- **memory-lancedb-pro-setup** -- one-click install/upgrade/repair script handling multiple scenarios (fresh install, git clone updates, config repair, npm update detection)
- **memory-lancedb-pro-skill** -- Claude Code/OpenClaw skill for AI-guided configuration with 7-step workflow and 4 deployment plans

## Funding & Sponsorship

- **No funding information** found in package.json
- **No sponsor button** configuration detected
- **No GitHub Sponsors** link visible
- **Maintained by individual contributors** -- primarily win4r as author

## Assessment Summary

| Dimension | Status | Notes |
|-----------|--------|-------|
| Activity Level | Very High | 376 commits in March 2026; active development |
| Contributor Base | Healthy | 50+ unique contributors; diverse geography |
| Documentation | Excellent | 11 README translations; comprehensive docs |
| Issue Templates | Good | Structured bug/feature templates with required fields |
| CI/CD | Basic | Version sync + test; no auto-publish |
| Release Process | Regular | Frequent stable and beta releases |
| Community Support | Discord-based | Active Discord community referenced |
| Funding | None | No sponsorship infrastructure |

## Strengths

1. **Exceptional internationalization** -- 11 language README translations
2. **Active development** -- very high commit velocity in 2026
3. **Diverse contributor base** -- strong Chinese-speaking developer representation
4. **Structured issue process** -- good issue templates with required fields
5. **Active beta program** -- v1.1.0-beta.x series for new features
6. **Community tooling ecosystem** -- setup scripts, configuration skills

## Areas for Improvement

1. **No CONTRIBUTING.md** -- a contributing guide would help onboard new contributors
2. **No PR template** -- PR template would improve code review quality
3. **No CODEOWNERS** -- would help auto-assign reviewers by path
4. **No funding/sponsorship** -- no way for users to support development
5. **No MAINTAINERS file** -- unclear who has merge rights or decision authority
6. **No auto-publish in CI** -- releases appear manual
