# Community Health Report: NanoClaw

## Repository Overview

| Field | Value |
|-------|-------|
| **Repository** | qwibitai/nanoclaw |
| **Default Branch** | main |
| **Total Commits** | 657 |
| **Latest Version** | 1.2.35 (2026-03-26) |
| **Release Model** | Semantic versioning with frequent releases |

## Activity Metrics

### Commit Activity

Recent commit activity shows **high velocity** with multiple commits daily:

```
2026-03-26: 5 commits (OneCLI Agent Vault migration)
2026-03-25: 15+ commits (PR merges, skill additions, version bumps)
2026-03-24: Active development
2026-03-23: Active development
2026-03-22: Active development
```

**Assessment:** Extremely active development with daily commits and multiple PR merges per day.

### Release Cadence

- **Frequency:** Multiple releases per week
- **Pattern:** Semantic versioning (MAJOR.MINOR.PATCH)
- **Recent releases:** 1.2.35, 1.2.34, 1.2.33 within the past 2 days
- **Changelog:** Maintained in CHANGELOG.md with links to full documentation
- **Breaking changes:** Clearly marked with `[BREAKING]` tag

## Contributor Landscape

### Top Contributors (by commit count)

| Rank | Contributor | Commits | Role |
|------|-------------|---------|------|
| 1 | gavrielc | 309 | Core maintainer |
| 2 | github-actions[bot] | 154 | Automated (CI/CD) |
| 3 | Gabi Simons | 29 | Core maintainer |
| 4 | glifocat | 20 | Contributor |
| 5 | Gavriel Cohen | 14 | Contributor |
| 6 | Koshkoshinsk | 13 | Contributor |
| 7 | NanoClaw User | 12 | Contributor |
| 8 | Claude | 10 | Automated/Contributor |
| 9 | Tom Granot | 9 | Contributor |
| 10 | Akshan Krithick | 5 | Contributor |

**Assessment:** Healthy contributor distribution with 2 core maintainers and 8+ regular external contributors.

## Governance Structure

### CODEOWNERS

```
# Core code - maintainer only
/src/              @gavrielc @gabi-simons
/container/        @gavrielc @gabi-simons
/groups/           @gavrielc @gabi-simons
/launchd/          @gavrielc @gabi-simons
/package.json      @gavrielc @gabi-simons
/package-lock.json @gavrielc @gabi-simons

# Skills - open to contributors
/.claude/skills/
```

**Assessment:** Clear separation between core code (maintainer-controlled) and skills (community-open).

## Community Infrastructure

### GitHub Workflows

| Workflow | Purpose |
|----------|---------|
| `ci.yml` | Continuous integration |
| `bump-version.yml` | Automated version bumps |
| `label-pr.yml` | Auto-label PRs by type |
| `update-tokens.yml` | Scheduled token updates |

### PR Template

Structured PR template with type classification:

| Type | Auto-Labels |
|------|-------------|
| Feature skill | `PR: Skill`, `PR: Feature` |
| Utility skill | `PR: Skill` |
| Operational/container skill | `PR: Skill` |
| Fix | `PR: Fix` |
| Simplification | `PR: Refactor` |
| Documentation | `PR: Docs` |

### Contributing Guidelines

Comprehensive CONTRIBUTING.md covering:
- Pre-submission checklist (search existing PRs/issues first)
- Four distinct skill types with clear definitions
- SKILL.md format requirements (under 500 lines)
- Testing requirements on fresh clone
- One-thing-per-PR policy

## Strengths

1. **Active development:** Multiple commits and releases daily
2. **Clear contribution path:** Well-documented skill taxonomy with separate code/branch ownership
3. **Good governance:** CODEOWNERS separates core (maintainer-only) from skills (community)
4. **Automated tooling:** PR labels, version bumps, CI all automated
5. **Contributor recognition:** Regular additions to contributors list
6. **Breaking change communication:** Clear [BREAKING] markers in changelog

## Areas for Improvement

1. **GitHub Releases not used:** Only git tags, no formal GitHub Releases with release notes
2. **External issue templates:** No issue templates found in .github/
3. **No MAINTAINERS file:** Governance structure exists only in CODEOWNERS
4. **No AUTHORS file:** Contributors tracked via git history only
5. **Limited public metrics:** Stars/forks/PRs not accessible via API (may be private or zero)

## Summary

| Dimension | Status |
|-----------|--------|
| **Development Activity** | Excellent - daily commits, weekly releases |
| **Contributor Base** | Good - 2 core + 8+ regular contributors |
| **Governance** | Good - clear separation core/skills |
| **Documentation** | Good - comprehensive CONTRIBUTING.md |
| **Automation** | Good - CI, auto-labels, version bumps |
| **Release Process** | Moderate - tags but no formal Releases |
| **Community Health** | **Positive** - active, welcoming, well-structured |

**Overall Assessment:** NanoClaw shows strong community health with active development, clear governance, and a well-structured contribution model. The skill-based architecture enables community participation without compromising core stability.
