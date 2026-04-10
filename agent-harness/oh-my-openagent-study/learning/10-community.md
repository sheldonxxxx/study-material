# Community: oh-my-openagent

## Repository Metrics

| Metric | Value |
|--------|-------|
| **Stars** | 43,738 |
| **Forks** | 3,241 |
| **Open Issues** | ~1 |
| **Open PRs** | ~1 |
| **Latest Release** | v3.13.1 (2026-03-25) |
| **Repository Created** | 2025-12-03 |
| **Last Pushed** | 2026-03-26 |
| **Total Commits** | 3,338+ |

## Activity Indicators

| Indicator | Status |
|-----------|--------|
| Recent commits | Very High (64 today, 907 this month) |
| Issue response | Excellent (~1 open issue) |
| PR throughput | High (1 open PR, active merges) |
| Release frequency | High (multiple releases per week) |
| External contributions | Yes (community contributors) |
| CLA enforcement | Yes |

## Top Contributors

| Rank | Contributor | Commits | Role |
|------|-------------|---------|------|
| 1 | YeonGyu-Kim | 2,501 | Core maintainer |
| 2 | github-actions[bot] | 443 | Automation |
| 3 | justsisyphus | 373 | Core team |
| 4 | Sisyphus | 78 | Core team |
| 5 | acamq | 77 | External contributor |
| 6 | Kenny | 69 | External contributor |
| 7 | MoerAI | 39 | External contributor |
| 8 | sisyphus-dev-ai | 37 | AI agent |
| 9 | Ravi Tharuma | 28 | External contributor |
| 10 | Junho Yeo | 22 | External contributor |

**Core team:** YeonGyu-Kim, justsisyphus, Sisyphus (appear to be the same person or tight team)

**Notable:** The project uses AI agents (Sisyphus, sisyphus-dev-ai) as contributors, reflecting its nature as an AI coding tool.

## Release Cadence

| Aspect | Value |
|--------|-------|
| Total releases | 60+ in v3.x line |
| March 2026 commits | 907 |
| Daily commits (avg) | 64+ |
| Release naming | Semantic versioning with codenames |

### Recent Releases

- v3.13.1 (2026-03-25) - Latest
- v3.5.0 - "Atlas Trusts No One"

### Community Contributions in Releases

Recent releases credit external contributors:
- @RaviTharuma: Claude model variant clamping fix
- @MoerAI: Multiple fixes including provider-agnostic fallback, config improvements

## Governance

| Aspect | Status |
|--------|--------|
| CODEOWNERS | Not present |
| MAINTAINERS | Not present |
| AUTHORS | Not present |
| CODE_OF_CONDUCT.md | Not present |
| FUNDING.yml | Present (GitHub sponsorship) |
| CLA | Enforced via cla.yml workflow |

### Funding

GitHub Sponsors link available for code-yeongyu (maintainer).

### CLA Enforcement

Contributor License Agreement workflow (`cla.yml`) runs on all PRs, requiring contributors to sign before merging.

## Community Infrastructure

### Discord

**Invite:** https://discord.gg/PUwSMR9XNk

Features:
- #building-in-public channel (live development with Jobdori AI)
- Real-time feature development
- Issue triage
- Community support

### Social Media

| Platform | Handle | Purpose |
|----------|--------|---------|
| X (Twitter) | @justsisyphus | News and updates |

### GitHub Assets

- Hero image (credit to @junhoyeo)
- Agent images (Sisyphus, Hephaestus)
- Building-in-public graphic

## Issue Templates

4 structured templates:
- **bug_report.yml** - Comprehensive with doctor output requirement
- **config.yml** - Configuration issues
- **feature_request.yml** - Feature requests
- **general.yml** - General issues

### Bug Report Requirements

- English language compliance
- Duplicate search
- Latest version confirmation
- Documentation verification
- Doctor output (`bunx oh-my-opencode doctor`)
- OS and version information

## PR Template

Structured template with:
- Summary (1-3 bullets)
- Changes (specific modifications)
- Screenshots (before/after)
- Testing (mandatory typecheck + test)
- Related issues (Closes # syntax)

## Contributing Guidelines

CONTRIBUTING.md covers:
- Language policy (English required)
- Prerequisites (Bun, TypeScript, OpenCode)
- Development setup
- Code style conventions
- Anti-patterns
- PR process

### Branch Strategy

- `dev` - Development branch (PR target)
- `master` - Stable (merged from dev on release)
- No direct PRs to master (blocked by CI)

## Public Building

The maintainer builds in public with Jobdori (AI assistant):
- Live development streams in Discord
- Every feature, fix, issue triage visible
- Encourages community participation

## Professional Adoption

Organizations using oh-my-openagent:
- Indent (Google, Microsoft references)
- ELESTYLE (Japanese payment solutions)
- Individual professionals across industries

## Community Health Assessment

| Dimension | Status |
|-----------|--------|
| Activity | Excellent |
| Response time | Fast (~1 open issue) |
| Release quality | High (detailed changelogs) |
| External contributions | Active |
| Documentation | Comprehensive |
| Governance clarity | Needs improvement |

## Areas for Improvement

| Issue | Impact | Priority |
|-------|--------|----------|
| No CODEOWNERS | Unclear ownership of areas | Medium |
| No MAINTAINERS file | Core team not documented | Medium |
| No CODE_OF_CONDUCT | Community standards not explicit | Medium |
| Docs not in-repo | External dependency risk | Low |

## Related Projects

The README references relationships with:
- **AmpCode** - Influence for features
- **Claude Code** - Feature inspiration
- **OpenCode** - The platform this plugin extends
- **Sisyphus Labs** - AI agent company (creator)

## License

**SUL-1.0** (Sisyphus Labs License 1.0)

Not a standard open-source license. Custom license from Sisyphus Labs.

---

**Summary:** Active, fast-moving project with strong community engagement, professional infrastructure, and visible public development. Core team drives most commits but actively integrates external contributions. Governance could be more explicit.
