# OASIS Community Health Report

## Repository Overview

- **Repository**: [camel-ai/oasis](https://github.com/camel-ai/oasis)
- **Description**: OASIS: Open Agent Social Interaction Simulations with One Million Agents
- **Stars**: 3,864
- **Forks**: 401
- **Open Issues**: ~9 (based on recent activity)
- **Open Pull Requests**: 9
- **Repository Topics**: agent-based-framework, agent-based-simulation, large-language-models, large-scale, llm-agents, deep-learning, natural-language-processing, ai-societies, multi-agent-systems

## Release Cadence and Versioning

- **Versioning**: Semantic Versioning (semver)
- **Release Model**: Active development; releases made when significant changes accumulate
- **Recent Releases**:
  - v0.2.5 (December 2025) - Update camel-ai to 0.2.78, Docker/Compose support, bug fixes
  - v0.2.3 (June 2025) - Added report post action for inappropriate content
  - v0.2.2 (June 2025) - Added Group Chat Action
  - v0.2.1 (June 2025) - Added interview action, changed twitter_channel to channel
  - v0.2.0 (May 2025) - Major update with PettingZoo-style interface, customizable agent models/tools/prompts
- **Release Assets**: Python wheel (.whl) and source tarball (.tar.gz) published to GitHub Releases

## Top Contributors

| Contributor | Contributions |
|------------|---------------|
| echo-yiyiyi | 393 |
| zhangzaibin | 88 |
| Konisberg | 81 |

The project shows a healthy contributor diversity with community members making meaningful contributions (e.g., Docker support, documentation improvements, bug fixes).

## Governance and Maintenance

- **CODEOWNERS**: Not present
- **MAINTAINERS**: Not present
- **AUTHORS**: Not present
- **Organization**: Part of [CAMEL-AI.org](https://github.com/camel-ai) GitHub organization
- **Sprint Planning**: 4-week sprints with biweekly reviews (documented in CONTRIBUTING.md)

## Contributing Infrastructure

### Issue Templates

The repository has structured GitHub issue templates:

1. **Bug Report** (`bug_report.yml`)
   - Requires reading documentation
   - Requires searching issue tracker/discussions
   - Captures version, system info, reproducible code, traceback
   - Labels: bug

2. **Feature Request** (`feature_request.yml`)
   - Requires searching tracker first
   - Encourages discussion before filing
   - Captures motivation, solution, alternatives
   - Labels: enhancement

3. **Questions** (`questions.yml`)
   - For user questions and discussions

4. **Discussions** (`discussions.yml`)
   - For general discussions

### Pull Request Template

The PR template is comprehensive and includes:
- Link to related issue (required)
- Checklist for code changes, tests, documentation, examples
- Special checklist for `SocialAgent` actions (ActionType in typing.py, tests, documentation)
- Pre-commit hook requirements (formatting/linting)

### GitHub Actions Workflows

- **python-app.yml**: CI pipeline for running tests

## Contributing Guidelines

The CONTRIBUTING.md is thorough and well-structured:

### Code Contribution Process
- Fork-and-Pull-Request workflow for community members
- Checkout-and-Pull-Request workflow for org members (to pass tests requiring GitHub Secrets)
- Requires passing formatting, linting, and testing checks

### Code Review Guidelines
- Minimum 1 reviewer approval required to merge
- Review checklist covering functionality, code quality, and design
- Constructive feedback requirement
- Timely review expectation

### Development Setup
```bash
poetry install
pre-commit install
poetry run pre-commit run --all-files
poetry run pytest
```

### Documentation
- Uses Mintlify for documentation
- Comprehensive docstring guidelines (Google Python Style Guide)
- Local doc building with `mintlify dev`

### PR Labeling Convention
- feat, fix, docs, style, refactor, test, chore

## Repository Activity

- **Commits since January 2025**: 467
- **Active Development**: Recent commits show regular activity (multiple times per week)
- **Maintenance**: Actively maintained with recent updates (latest commit: March 2026)

## Community Engagement

- **Communication Channels**:
  - Discord: Active channel for developer discussions
  - WeChat: Group for Chinese-speaking community
  - Developer Meetings: Weekly meetings (Chinese speakers: Thursday 10 PM UTC+8)
- **New Contributor Recognition**: Release notes credit first-time contributors
- **Response to Issues**: Active issue triage with labels and project boards

## Assessment

| Dimension | Status |
|-----------|--------|
| Release Management | Healthy - regular semantic versioning releases |
| Contributor Diversity | Good - 3+ active contributors beyond core team |
| Documentation | Excellent - comprehensive CONTRIBUTING.md, issue templates |
| CI/CD | Basic - GitHub Actions workflow present |
| Governance | Minimal - no CODEOWNERS/MAINTAINERS/AUTHORS files |
| Community Engagement | Strong - multiple communication channels, meeting schedule |
| Issue Response | Active - regular updates, project board workflow |
