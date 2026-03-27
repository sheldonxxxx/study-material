# Community Health Analysis: Superpowers

## Repository Overview

**Description:** Superpowers is a complete software development workflow for coding agents, built on composable "skills" and instructions that guide agents through specification, planning, and implementation.

## Engagement Metrics

| Metric | Value |
|--------|-------|
| Latest Release | v5.0.6 (March 25, 2026) |
| Total Commits (since ~Oct 2025) | 407 in 2025-2026 |
| Top Contributor | Jesse Vincent (483 commits) |
| Active Contributors | 5+ (Jesse Vincent, Drew Ritter, CL Kao, Joshua Shanks, others) |

**Note:** GitHub API (gh) not accessible from this environment; stars/forks counts unavailable.

## Release Cadence

The project uses semantic versioning with frequent releases:
- v5.0.x series currently active (v5.0.6 latest as of March 2026)
- Historical releases: v4.3.1, v4.3.0, v4.2.0, v4.1.1
- Release cycle appears rapid with 407 commits spanning ~5 months (Oct 2025 - Mar 2026)
- Approximately 1-2 releases per month based on tag frequency

## Contribution Guidelines

### Pull Request Template

The project has a detailed PR template requiring:

1. **Problem statement** - Specific problem encountered with failure mode
2. **Change description** - 1-3 sentences on what changed
3. **Core library fit** - Justification that change benefits general users (not project-specific)
4. **Alternatives considered** - Other approaches evaluated
5. **Unrelated changes check** - Multiple unrelated changes must be split
6. **Duplicate PR check** - Review of open/closed PRs required
7. **Environment tested** - Harness, version, model information
8. **Evaluation results** - Before/after comparison across multiple sessions
9. **Adversarial testing** - Skills changes require `superpowers:writing-skills` and pressure testing
10. **Human review** - Checkbox confirming human reviewed complete diff

**Filtering:** PRs with blank sections, multiple unrelated changes, or no evidence of human involvement are closed without review.

### Issue Templates

Three issue templates provided:
- **Bug Report** - Requires environment info, platform vs Superpowers distinction, reproduction steps, debug log
- **Feature Request** - Requires problem statement, proposed solution, alternatives, core fit justification
- **Platform Support** - For platform-specific issues

### Governance Files

- **FUNDING.yml** - GitHub Sponsors available for author `obra`
- **No CODEOWNERS** - No code ownership file
- **No MAINTAINERS** - No formal maintainer list
- **No AUTHORS** - No AUTHORS file

## Activity Assessment

| Indicator | Status |
|-----------|--------|
| Recent commits | Active (March 2026) |
| Release frequency | High (monthly+) |
| Contributor diversity | Low (1 dominant contributor) |
| Community engagement | Moderate (detailed PR template, issue templates) |
| Funding mechanism | GitHub Sponsors |

## Observations

1. **Single-maintainer project** - Jesse Vincent accounts for ~95% of commits
2. **High-quality contribution requirements** - Detailed templates discourage low-quality submissions
3. **Active development** - 407 commits in ~5 months indicates vigorous maintenance
4. **Clear standards** - PR template establishes expectations for evaluation and rigor
5. **Selective scope** - Strong emphasis on changes being appropriate for "core library" vs plugins

## Conclusion

Superpowers is a actively maintained single-maintainer project with high standards for contributions. The extensive PR template and issue templates indicate a focus on quality over quantity. The funding setup (GitHub Sponsors) suggests sustainable open-source maintenance model. Contributor diversity is low but contribution quality requirements are high.
