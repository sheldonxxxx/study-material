# CoPaw Community

## Overview

CoPaw is an Apache 2.0 licensed open-source project under the **AgentScope** organization. The project demonstrates strong community infrastructure with bilingual documentation and active development.

## Repository Information

| Attribute | Value |
|-----------|-------|
| **Repository** | `agentscope-ai/CoPaw` |
| **License** | Apache License 2.0 |
| **Latest Version** | v0.2.0 (2026-03-24/25) |
| **GitHub Stars** | Available on repo badge |
| **GitHub Forks** | Available on repo badge |

## Release Cadence

### Version History
- **v0.2.0** (2026-03-24/25) - Major release with inter-agent communication, QA agent, audio/video input
- **v0.1.0** (2026-03-18) - Feature release
- **v0.0.7** (2026-03-12)
- **v0.0.6** (2026-03-09)
- **v0.0.5** (2026-03-06)
- **v0.0.4** (2026-03-02)

### Release Types
| Type | Pattern | Example |
|------|---------|---------|
| Stable | `v0.0.x` | v0.2.0 |
| Beta | `v0.0.x-beta.N` | v0.1.0-beta.1 through beta.4 |
| Post | `.post1` suffix | Quick patches |

### Activity Level
- **Very High**: 30+ commits in 3 days (recent sample)
- Daily commits across multiple contributors
- Feature releases every 1-2 weeks

## Contributing Infrastructure

### Contributing Guidelines
- **Location**: `.github/CONTRIBUTING.md`
- **Quality**: Comprehensive
- **Languages**: English + Chinese (`CONTRIBUTING_zh.md`)
- **Key elements**:
  - Step-by-step contribution process
  - Conventional Commits specification
  - Required local gate: `pre-commit run --all-files` and `pytest`
  - PR title format requirements
  - Component categorization

### Issue Templates

| Template | Purpose |
|----------|---------|
| `1-question.md` | General questions |
| `2-feature_request.md` | Feature proposals |
| `3-documentation.md` | Documentation improvements |
| `4-bug_report.md` | Bug reports with detailed structure |
| `5-support_environment.md` | Environment-related issues |

**Bug report includes:**
- Version information
- Component affected
- Environment (OS, Python version, install method)
- Steps to reproduce
- Expected vs actual behavior
- Log/screenshot capture

### Pull Request Template

**Location**: `.github/PULL_REQUEST_TEMPLATE.md`

**Includes:**
- Description field
- Related issue reference
- Security considerations
- Change type checklist
- Component affected checklist
- Pre-commit and test verification evidence
- Local verification instructions

### Security Policy

**Location**: `SECURITY.md`
**Quality**: Highly detailed, professional

**Key elements:**
- Private disclosure via Alibaba Security Response Center (ASRC)
- Detailed reporting requirements
- Clear trust model (single-operator boundary)
- Out of scope items defined
- No bug bounty program
- Trusted Skills concept explained

## Commit Convention

CoPaw follows **Conventional Commits**:

```
<type>(<scope>): <subject>
```

**Types:**
- `feat` - New feature
- `fix` - Bug fix
- `docs` - Documentation
- `style` - Formatting
- `refactor` - Refactoring
- `perf` - Performance
- `test` - Tests
- `chore` - Maintenance

**Examples:**
```
feat(channels): add Telegram channel stub
fix(skills): correct SKILL.md front matter parsing
docs(readme): update quick start for Docker
```

## Maintainer Structure

### Files NOT Present
- **CODEOWNERS** - Not present
- **MAINTAINERS** - Not present
- **AUTHORS** - Not present

### Governance Indicators
- Apache 2.0 license (permissive, corporate-friendly)
- Clear contributor guidelines
- Professional security policy
- Active development suggests engaged maintainers
- ASRC partnership for security

## Community Channels

| Channel | Purpose |
|---------|---------|
| **GitHub Discussions** | General discussion, Q&A |
| **GitHub Issues** | Bug reports, feature requests |
| **Discord** | Real-time chat (link in README) |
| **DingTalk** | Chinese community (link in README) |
| **X (Twitter)** | Updates (@agentscope_ai) |

## CI/CD Automation

**Location**: `.github/workflows/`
**Count**: 12 workflow files

**Automated:**
- Testing (multi-platform)
- Pre-commit validation
- Docker builds
- Desktop builds
- PyPI publishing
- Website deployment
- Welcome messages for issues/PRs

## Community Health Assessment

| Indicator | Status | Notes |
|-----------|--------|-------|
| Contributing guide | Excellent | Comprehensive with examples, Chinese translation |
| Issue templates | Good | Multiple types, well-structured |
| PR template | Excellent | Detailed verification requirements |
| Security policy | Excellent | ASRC partnership, clear trust model |
| Multi-language docs | Good | English + Chinese |
| Release tags | Active | Regular semantic versioning |
| Commit activity | Very High | Daily commits |
| Response to issues | Unknown | Metrics unavailable |

## Contribution Areas

Per README and CONTRIBUTING:

### Seeking Contributors
- Horizontal expansion (new channels, models, skills, MCPs)
- Existing feature extension (display, UX, Windows compatibility)

### In Progress (Community Can Help)
- Feature extension and optimization
- Documentation improvements

### Core Team Focus
- Multi-agent system
- Self-healing capabilities
- Multimodal (voice/video)
- Memory system
- Sandbox integration

## Getting Started for Contributors

```bash
# 1. Fork and clone
git clone https://github.com/YOUR_FORK/CoPaw.git
cd CoPaw

# 2. Install dev environment
pip install -e ".[dev,full]"
pre-commit install

# 3. Run local gate
pre-commit run --all-files
pytest

# 4. Make changes
# - Follow conventional commits
# - Update docs if user-facing
# - Add tests if applicable

# 5. Open PR
# - Reference related issue
# - Pass all CI checks
# - Update docs for user-facing changes
```

## Contributor Recognition

Recent v0.2.0 contributors acknowledged:
- @ixiadao, @leoleils, @ltzu929, @emoubarak, @f3125472, @shiweijiezero, @Yaohua-Leo, @finenter-molei, @lizeruicq, @hbsjwj, @aquamarine-bot, @sanfran1068, @x1n95c, @saschabuehrle

## Documentation Translations

| Language | README | Contributing | Docs |
|----------|--------|--------------|------|
| English | README.md | CONTRIBUTING.md | *.en.md |
| Chinese | README_zh.md | CONTRIBUTING_zh.md | *.zh.md |
| Japanese | README_ja.md | - | - |
| Russian | - | - | Agent prompts only |

## Strengths

1. **Professional security policy** with ASRC partnership
2. **Comprehensive contributing guidelines** with Chinese translation
3. **Detailed PR and issue templates**
4. **Active development** with daily commits
5. **Multi-language documentation** (EN, ZH, JA, RU for prompts)
6. **Clear conventional commit convention**
7. **Strong CI/CD infrastructure**

## Potential Gaps

1. **No public MAINTAINERS file** - Governance less visible
2. **No CODEOWNERS** - Review assignment less structured
3. **Community metrics unavailable** - Stars/forks/response time not accessible via gh CLI
4. **No official community forum** - Only GitHub Discussions and chat platforms
5. **Release notes are complete but lack migration guides** for breaking changes
