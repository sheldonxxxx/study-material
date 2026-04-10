# Community

## Overview

OASIS is developed and maintained by CAMEL-AI.org with an active open-source community contributing features, bug reports, and research extensions.

## Project Affiliation

- **Organization**: CAMEL-AI.org
- **GitHub**: https://github.com/camel-ai/oasis
- **Repository Star History**: Active growth, notable spike around November 2024 (initial release)

## Communication Channels

| Channel | Purpose | Link |
|---------|---------|------|
| Discord | General discussion, questions | https://discord.camel-ai.org/ |
| WeChat | Chinese-speaking community | QR code in README |
| GitHub Issues | Bug reports, feature requests | https://github.com/camel-ai/oasis/issues |
| GitHub PRs | Code contributions | https://github.com/camel-ai/oasis/pulls |
| Email | Direct contact | camel.ai.team@gmail.com |

## Developer Meetings

- **Chinese Speakers**: Thursdays at 10 PM UTC+8 via Tencent Meeting
- **English Speakers**: Coming soon (as of documentation)

## Contribution Workflow

### For Community Contributors
Follow Fork-and-Pull-Request workflow:
1. Fork the repository
2. Create a feature branch
3. Make changes
4. Submit pull request

### For CAMEL-AI Members
Follow Checkout-and-Pull-Request workflow (required for GitHub Secrets access)

## Contribution Requirements

### Code Quality Gates
All PRs must pass:
- Pre-commit checks (formatting, linting, license headers)
- pytest test suite
- Code review by at least one maintainer

### Testing Requirements
| Change Type | Required Action |
|------------|-----------------|
| Bug fix | Add unit test if possible |
| Improvement | Update examples + documentation + tests |
| New feature | Include unit tests + demo script |

### Documentation Standards
- Google Python Style Guide for docstrings
- Raw docstrings (`r"""`)
- 79-character line limit
- Comprehensive Args sections

## Code Review Process

1. Reviewer checks functionality, readability, consistency
2. Feedback provided if changes needed
3. Contributor addresses feedback
4. Reviewer re-reviews
5. At least one approval required for merge
6. Maintainer performs merge

### Review Checklist
- Correctness and edge case handling
- Test coverage and passing tests
- Security considerations
- Performance implications
- Readability and maintainability
- Style compliance (Ruff, Google Style)
- Documentation completeness

## Project Governance

### Sprint Cycle
- **Duration**: 4 weeks
- **Planning & Review**: Biweekly during dev meeting (~30 minutes)
- **Sprint Goal**: Founder defines; developers pick items

### Issue Management
1. Create issue with category (bug, improvement, feature)
2. Fill required fields (title, assignees, labels, projects, milestones)
3. Discuss in team meetings
4. Move through: Backlog -> Analysis Done -> Sprint Planned -> Developing -> Reviewing -> Merged

### PR Labels
- `feat` - New features
- `fix` - Bug fixes
- `docs` - Documentation
- `style` - Formatting changes
- `refactor` - Code restructuring
- `test` - Tests
- `chore` - Maintenance

## Research Extensions

The project tracks follow-up research built on OASIS:
- [MultiAgent4Collusion](https://github.com/renqibing/MultiAgent4Collusion) - Collusion simulation
- [CUBE](https://github.com/echo-yiyiyi/cube) - Unity3D environment simulations
- [MultiAgent4Fraud](https://github.com/zheng977/MutiAgent4Fraud) - Fraud detection research

## Citation

```bibtex
@misc{yang2024oasisopenagentsocial,
  title={OASIS: Open Agent Social Interaction Simulations with One Million Agents},
  author={Ziyi Yang et al.},
  year={2024},
  eprint={2411.11581},
  archivePrefix={arXiv},
  primaryClass={cs.CL}
}
```

## Acknowledgments

- Logo designed by Douglas
- Contributors acknowledged via GitHub

## License

Apache 2.0 - All source code, contributions become Apache 2.0 licensed

## Community Health

- **Responsiveness**: Maintainers available via Discord, WeChat, email
- **Documentation**: Comprehensive contributing guide with detailed workflows
- **Transparency**: Public sprint planning, regular updates in README changelog
- **Inclusivity**: Bilingual support (English + Chinese), multiple communication channels
