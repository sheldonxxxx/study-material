# Community Health Report: oh-my-openagent

## Repository Overview

| Metric | Value |
|--------|-------|
| Stars | 43,738 |
| Forks | 3,241 |
| Open Issues | ~1 |
| Open PRs | ~1 |
| Latest Release | v3.13.1 (2026-03-25) |
| Created | 2025-12-03 |
| Last Pushed | 2026-03-26 |

## Repository Metadata

- **Description**: "omo; the best agent harness - previously oh-my-opencode"
- **Topics**: claude-code, opencode, ai, anthropic, claude, cursor, gemini, ide, openai, orchestration, tui, typescript, ai-agents, chatgpt

## Governance Files

| File | Status |
|------|--------|
| CODEOWNERS | Not present |
| MAINTAINERS | Not present |
| AUTHORS | Not present |
| CODE_OF_CONDUCT.md | Not present |
| FUNDING.yml | Present (GitHub sponsorship link for code-yeongyu) |

## Contributing Infrastructure

### .github/ Contents

```
.github/
  FUNDING.yml          # GitHub sponsorship configured
  ISSUE_TEMPLATE/
    bug_report.yml     # Detailed bug report with doctor output
    config.yml
    feature_request.yml
    general.yml
  pull_request_template.md  # Structured PR template with testing section
  workflows/           # CI/CD pipelines (7 workflow files)
    ci.yml
    cla.yml
    lint-workflows.yml
    publish-platform.yml
    publish.yml
    refresh-model-capabilities.yml
    sisyphus-agent.yml
  assets/
```

### Issue Templates

The repository has 4 issue templates:
- **bug_report.yml** - Comprehensive template requiring doctor output, reproduction steps, OS, and version
- **config.yml** - Configuration issues
- **feature_request.yml** - Feature requests
- **general.yml** - General issues

Bug reports require:
- English language compliance
- Search for duplicates
- Latest version confirmation
- Documentation verification
- Doctor output (`bunx oh-my-opencode doctor`)
- OS and version information

### PR Template

The PR template includes:
- Summary section (1-3 bullet points)
- Changes section (specific modifications)
- Screenshots section (before/after)
- Testing section (mandatory `bun run typecheck` and `bun test`)
- Related Issues section (with Closes # syntax)

## Contributing Guidelines

**CONTRIBUTING.md** exists with comprehensive documentation covering:
- Language Policy: English required for all communications
- Prerequisites (Bun, TypeScript 5.7.3+, OpenCode 1.0.150+)
- Development setup and testing
- Project structure
- Build commands and code style
- Pull request process

## Top Contributors (by commits since Dec 2025)

| Contributor | Commits |
|-------------|---------|
| YeonGyu-Kim | 2,501 |
| github-actions[bot] | 443 |
| justsisyphus | 373 |
| Sisyphus | 78 |
| acamq | 77 |
| Kenny | 69 |
| MoerAI | 39 |
| sisyphus-dev-ai | 37 |
| Ravi Tharuma | 28 |
| Junho Yeo | 22 |

**Note**: The project appears to have a core team (YeonGyu-Kim, justsisyphus, Sisyphus) with external contributors (MoerAI, Ravi Tharuma, acamq, Kenny, Junho Yeo).

## Release Cadence

**Active release schedule** with 60+ releases in the v3.x line:
- Latest: v3.13.1 (2026-03-25)
- 64 commits in the last day (2026-03-25-26)
- 907 commits in March 2026
- 3,338 commits in 2025-2026

Recent releases include semantic versioning with notable names:
- v3.5.0 — "Atlas Trusts No One"

## GitHub Actions Integration

**CLA workflow** present, indicating Contributor License Agreement requirement for external contributors.

Recent community contributions in v3.13.1 release:
- @RaviTharuma: Claude model variant clamping fix
- @MoerAI: Multiple fixes including provider-agnostic fallback, config improvements

## Activity Assessment

| Indicator | Status |
|-----------|--------|
| Recent commits | Very High (64 today, 907 this month) |
| Issue response | Excellent (~1 open issue) |
| PR throughput | High (1 open PR, active merges) |
| Release frequency | High (multiple releases per week) |
| External contributions | Yes (community contributors acknowledged) |
| CLA enforcement | Yes (cla.yml workflow) |

## Summary

**Community Health: Excellent**

- Very active development with daily commits
- Professional infrastructure (PR templates, issue templates, CLA)
- Clear contributing guidelines with language policy
- Active external contributor community
- Transparent release notes crediting community members
- Fast issue/PR turnaround (~1 open each)
- Strong versioning discipline with semantic releases

**Areas for Improvement**:
- No CODEOWNERS file (could clarify maintainership)
- No MAINTAINERS file (core team not formally documented)
- No CODE_OF_CONDUCT.md file
