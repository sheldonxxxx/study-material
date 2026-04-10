# Community

## Overview

CrewAI has grown from a solo project to a significant open-source ecosystem with enterprise backing, substantial community engagement, and an active development trajectory.

## GitHub Metrics

Based on the README badges and repository data:

| Metric | Value | Source |
|--------|-------|--------|
| **GitHub Stars** | Significant (badge displayed) | `![GitHub Repo stars](https://img.shields.io/github/stars/crewAIInc/crewAI)` |
| **Forks** | Significant (badge displayed) | `![GitHub forks](https://img.shields.io/github/forks/crewAIInc/crewAI)` |
| **Watchers** | Active subscription | GitHub UI |
| **Issues/PRs** | Tracked via GitHub | `![GitHub issues](https://img.shields.io/github/issues/crewAIInc/crewAI)` |

*Note: Exact numbers not extracted; badges render dynamically*

## Growth Indicators

### Developer Certification

> "Over 100,000 developers certified through our community courses at learn.crewai.com"

### Enterprise Adoption

- **AMP Suite (Advanced Multi-Agent Platform)**: Enterprise offering with control plane, observability, and support
- **100,000+ certified developers** indicates significant individual adoption

### Learning Platform

**learn.crewai.com** provides:
- Multi AI Agent Systems with CrewAI (DeepLearning.ai course)
- Practical Multi AI Agents and Advanced Use Cases (DeepLearning.ai course)
- Community certification program

## Governance

### Leadership

**Founder/Primary Maintainer:**
- Joao Moura (`joao@crewai.com`, `joaomdmoura@gmail.com`)

### Organizational Structure

- **Company**: crewAI Inc.
- **Repository**: github.com/crewAIInc/crewAI
- **Website**: crewai.com
- **Documentation**: docs.crewai.com
- **Community Forum**: community.crewai.com
- **Blog**: blog.crewai.com

### Project Ownership

The repository is maintained by `crewAIInc` organization with:
- Founder as primary maintainer
- Likely additional core contributors (not visible in repo metadata)
- Enterprise team for AMP Suite

## Release Cadence

### Version History

From pyproject.toml and changelog:
- Current documented version: v1.12.2 (in docs.json)
- Release candidates: `1.13.0rc1` (in tools dependency)
- Nightly releases: Daily at 6am UTC with `.dev{YYYYMMDD}` suffix

### Release Types

| Type | Frequency | Mechanism |
|------|-----------|-----------|
| **Stable Release** | On-demand | Manual `publish.yml` workflow |
| **Nightly Canary** | Daily (if changes) | Automated `nightly.yml` |
| **RC (Release Candidate)** | Per feature cycle | Manual dispatch |
| **Patch releases** | As needed | Manual dispatch |

### Release Automation

1. **Build**: `uv build --all-packages`
2. **Publish**: uv to PyPI with OIDC token
3. **Notify**: Slack integration with changelog parsing
4. **Version Source**: `__version__` in `__init__.py` files

## Contributor Experience

### Contributing Guidelines

Comprehensive CONTRIBUTING.md with:
- AI-generated contribution disclosure requirement (`llm-generated` label)
- Setup instructions with `uv` workflow
- Repository structure explanation
- Code style requirements (strict typing, ruff, mypy)
- Conventional commit enforcement
- Testing requirements
- Documentation translation policy

### AI Contribution Policy

Notable policy for AI-generated contributions:

> "If your PR or issue was authored by an AI agent, coding assistant, or LLM (e.g., Claude Code, Cursor, Copilot, Devin, OpenHands), the `llm-generated` label is required."

This indicates:
1. Acknowledgment of AI tool usage in OSS
2. Transparency requirement
3. Quality control for AI-authored changes

### Pre-commit Requirements

Contributors must:
1. Install pre-commit hooks: `uv run pre-commit install`
2. Pass ruff, mypy, and commitizen checks locally
3. Follow conventional commit format
4. Keep `uv.lock` synchronized

## Community Channels

### Official Channels

| Channel | URL | Purpose |
|---------|-----|---------|
| **Documentation** | docs.crewai.com | Official docs (Mintlify) |
| **Community Forum** | community.crewai.com | Q&A, discussions |
| **Blog** | blog.crewai.com | Announcements, tutorials |
| **GitHub Issues** | github.com/crewAIInc/crewAI/issues | Bug reports, features |
| **GitHub Discussions** | github.com/crewAIInc/crewAI/discussions | RFC, Q&A |

### Social Presence

| Platform | Handle | Link |
|----------|--------|------|
| **Twitter/X** | @crewAIInc | `![Twitter Follow](https://img.shields.io/twitter/follow/crewAIInc?style=social)` |
| **YouTube** | crewAI channel | Tutorial videos |
| **ChatGPT Plugin** | CrewGPT | g/g-qqTuUWsBY-crewai-assistant |

### Learning & Education

- **DeepLearning.ai Partnership**: Two official short courses
- **Community Certification**: 100,000+ certified developers
- **Examples Repository**: `crewAI-examples` with landing_page_generator, trip_planner, stock_analysis

## Communication Standards

### Issue Templates

GitHub issue templates available:
- Bug Report template
- Feature Request template

### Response Patterns

- Active issue management (stale workflow enabled)
- Size labels auto-applied to PRs (`size/XL` for >500 lines)
- Semantic PR title validation

## Diversity & Inclusion

### Internationalization

Documentation in 4 languages:
- English (default)
- Arabic
- Korean
- Brazilian Portuguese

This indicates commitment to global accessibility.

## Commercial Ecosystem

### Enterprise Offering: AMP Suite

```
CrewAI AMP Suite
├── Crew Control Plane (free tier available)
├── Tracing & Observability
├── Unified Control Plane
├── Enterprise Integrations
├── Advanced Security
├── 24/7 Support
└── On-premise and Cloud Deployment
```

### Revenue Model

1. **Open Source**: Core framework (MIT license)
2. **Enterprise**: AMP Suite with advanced features
3. **Cloud**: crewai.com platform
4. **Learning**: Certification programs

## Contributing Demographics

Based on CONTRIBUTING.md's AI policy, the project anticipates:
- Human contributors
- AI-assisted contributors (requiring disclosure)
- Corporate contributors (enterprise integration)

## External Recognition

### Comparison with Competitors

README positions CrewAI against:
- LangGraph (claims 5.76x faster execution)
- AutoGen
- ChatDev

### Benchmark Claims

- Performance advantage over LangGraph in QA tasks
- Higher evaluation scores with faster completion in coding tasks

## License

MIT License (per LICENSE file and README)

## Repository Health Indicators

### Activity

- Active workflows (tests, linting, type-checking on PRs)
- Nightly release automation
- Regular documentation updates
- Issue/PR management (stale workflow)

### Code Review

- Required PR title validation
- Size-based labeling
- Semantic PR enforcement
- LLM-generated label requirement

### Documentation

- Multi-language support
- Regular content updates
- Translation parity requirement in CONTRIBUTING.md

## Best Practices Established

1. **Conventional Commits**: Strict enforcement via PR title validation
2. **Type Safety**: Strict mypy with pydantic plugin
3. **Code Quality**: Ruff with extensive rule set
4. **Testing**: Comprehensive pytest with parallelization
5. **Dependency Management**: uv with lock file commitment
6. **AI Transparency**: Required disclosure of AI-generated contributions
