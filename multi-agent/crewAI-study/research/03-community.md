# crewAI Community Health Assessment

**Repository:** crewAIInc/crewAI
**Assessment Date:** 2026-03-27
**Data Sources:** GitHub API, local repository inspection

---

## Overview

| Metric | Value |
|--------|-------|
| Stars | 47,321 |
| Forks | 6,397 |
| Open Issues | 90 |
| Open Pull Requests | 371 |
| Latest Stable Release | v1.12.2 (2026-03-26) |
| Repository Topics | agents, ai, ai-agents, llms, aiagentframework |

---

## Repository Structure

The project uses a **uv workspace** monorepo structure with four packages under `lib/`:

| Package | Path | Description |
|---------|------|-------------|
| `crewai` | `lib/crewai/` | Core framework |
| `crewai-tools` | `lib/crewai-tools/` | Tool integrations |
| `crewai-files` | `lib/crewai-files/` | File handling |
| `devtools` | `lib/devtools/` | Internal release tooling |

Documentation lives in `docs/` with translations under `docs/{en,ar,ko,pt-BR}/`.

---

## Release Cadence

The project maintains a **rapid release cycle** with frequent version bumps:

- **v1.13.x** is in pre-release (rc1 as of 2026-03-27)
- **v1.12.x** is the current stable series (1.12.2 released 2026-03-26)
- Multiple alpha/beta pre-releases between stable versions
- Recent releases approximately every few days to weeks

This indicates **active development** with a fast-moving project.

---

## Top Contributors

| Rank | Contributor | Role |
|------|-------------|------|
| 1 | joaomdmoura | Primary maintainer (570+ contributions) |
| 2 | greysonlalonde | Core contributor, release management |
| 3 | (others via contribution graph) | Community contributors |

The **contribution graph is highly concentrated** around a small number of core maintainers, which is typical for AI framework projects but may indicate limited external core contributions.

---

## Contributing Infrastructure

### Issue Templates

The repository has structured issue templates:

- **Bug Report** (`bug_report.yml`) - Comprehensive with fields for OS, Python version, crewAI version, crewAI Tools version, virtual environment, steps to reproduce, expected vs actual behavior
- **Feature Request** (`feature_request.yml`)
- **Blank issues disabled** - Forces use of templates

### Pull Request Templates

PR title must follow Conventional Commits format (`<type>(<scope>): <description>`).

### GitHub Actions Workflows (13 total)

Active CI/CD infrastructure including:
- `linter.yml` - Code linting
- `pr-size.yml` - PR size tracking
- `pr-title.yml` - PR title validation
- `nightly.yml` - Nightly builds
- `docs-broken-links.yml` - Documentation validation
- `codeql.yml` - Security scanning
- `dependabot.yml` - Dependency updates
- `publish.yml` - Package publishing
- `stale.yml` - Stale issue management

### Pre-commit Hooks

Project enforces pre-commit hooks for code quality:
- `ruff` for linting and formatting
- `mypy` for type checking
- Tests must pass before commits

---

## Code Quality Standards

The project enforces strict standards:

- **Type annotations**: Full annotations required on all functions, methods, and classes
- **Python versions**: Supports 3.10-3.14 (development targets 3.12)
- **Docstrings**: Google-style, minimal but informative
- **Imports**: Use `collections.abc` for abstract base classes
- **Type narrowing**: Use `isinstance`, `TypeIs`, or `TypeGuard` instead of `hasattr`
- **No bare dict/list**: Must use generic syntax (`list[str]`, not `List[str]`)

### CI Type Checking

Mypy runs on Python 3.10, 3.11, 3.12, and 3.13 for every PR.

---

## Security

- **Security policy**: Bugcrowd VDP submission via crewai-vdp-ess@submit.bugcrowd.com
- **No public vulnerability disclosure** via GitHub issues
- **CodeQL** integration for automated security scanning

---

## Community Policies

### AI-Generated Contributions

Notable policy requiring AI agents to apply the `llm-generated` label:

> "If your PR or issue was authored by an AI agent, coding assistant, or LLM (e.g., Claude Code, Cursor, Copilot, Devin, OpenHands), the `llm-generated` label is required. This applies to code, documentation, and issues alike. Unlabeled AI-generated contributions may be closed without review."

This is a **progressive policy** acknowledging the prevalence of AI-assisted contributions.

---

## Missing Governance Files

The following common governance files are **absent**:
- `CODEOWNERS` - No explicit code ownership
- `MAINTAINERS` - No documented maintainer hierarchy
- `AUTHORS` - No author attribution file
- `.github/FUNDING.yml` - No funding configuration

This may indicate informal governance structure.

---

## Summary Assessment

| Dimension | Status | Notes |
|-----------|--------|-------|
| Activity | Excellent | 47k stars, active releases every few days |
| Contributor Diversity | Moderate | Concentrated around 2-3 core maintainers |
| Documentation | Strong | Multi-language docs, comprehensive contributing guide |
| CI/CD | Excellent | 13 workflows, automated quality gates |
| Governance | Informal | No CODEOWNERS, MAINTAINERS, or AUTHORS files |
| Security | Good | Bugcrowd VDP, CodeQL, security policy |
| Openness | Good | 371 open PRs, structured contribution process |

---

## Recommendations

1. **Consider formalizing maintainer structure** with a MAINTAINERS file
2. **Add CODEOWNERS** for clear code ownership
3. **Diversify core contributors** - High star count but concentrated commits
4. **Consider FUNDING.yml** to enable GitHub Sponsors
