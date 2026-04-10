# Community Health Analysis: nanobot

## Repository Overview

| Metric | Value |
|--------|-------|
| **Stars** | 36,324 |
| **Forks** | 6,220 |
| **Open Issues** | 817 |
| **Repository URL** | https://github.com/HKUDS/nanobot |
| **Created** | 2026-02-01 |
| **Last Push** | 2026-03-26 |

**Note:** The repository was created on 2026-02-01 under the HKUDS organization. While the commit history is recent, the project has an established version history with tags dating back through the v0.1.x series, indicating this may be a recent fork or transfer to the HKUDS organization with preserved version history.

---

## Maintainership Structure

### Primary Maintainers

| Maintainer | Role | Focus |
|------------|------|-------|
| [@re-bin](https://github.com/re-bin) | Project Lead | `main` branch |
| [@chengyongru](https://github.com/chengyongru) | Experimental Lead | `nightly` branch, experimental features |

### Branching Strategy

The project uses a **two-branch model**:

| Branch | Purpose | Stability |
|--------|---------|-----------|
| `main` | Stable releases | Production-ready |
| `nightly` | Experimental features | May have bugs or breaking changes |

**Cherry-pick workflow:** Stable features are cherry-picked from `nightly` into `main` approximately weekly.

---

## Top Contributors

Based on commit history (top 10 by commits):

| Rank | Contributor | Commits |
|------|-------------|---------|
| 1 | Re-bin | 677 |
| 2 | Xubin Ren | 197 |
| 3 | chengyongru | 66 |
| 4 | Alexander Minges | 47 |
| 5 | chaohuang-ai | 21 |
| 6 | Nikolas de Hor | 20 |
| 7 | Sergio Sanchez Valles | 16 |
| 8 | flobo3 | 12 |
| 9 | pinhua33 | 12 |
| 10 | Kunal Karmakar | 10 |

**Total unique contributors:** 200+

---

## Contributing Guidelines

### Development Setup

```bash
git clone https://github.com/HKUDS/nanobot.git
cd nanobot
pip install -e ".[dev]"
pytest
ruff check nanobot/
ruff format nanobot/
```

### Code Standards

- **Python:** 3.11+
- **Linter:** ruff (rules E, F, I, N, W; E501 ignored)
- **Line length:** 100 characters
- **Async:** uses `asyncio` throughout
- **Testing:** pytest with `asyncio_mode = "auto"`

### Branch Targeting

| Change Type | Target Branch |
|-------------|---------------|
| New feature | `nightly` |
| Bug fix | `main` |
| Documentation | `main` |
| Refactoring | `nightly` |
| Unsure | `nightly` |

### Code Quality Values

The project emphasizes:
- **Simple:** smallest change that solves the real problem
- **Clear:** optimized for the next reader
- **Decoupled:** clean boundaries, no unnecessary abstractions
- **Honest:** no hidden complexity
- **Durable:** easy to maintain, test, and extend

---

## Security Policy

The project has a comprehensive [SECURITY.md](https://github.com/HKUDS/nanobot/blob/main/SECURITY.md) covering:

- **Private vulnerability reporting** via GitHub Security Advisories or email (xubinrencs@gmail.com)
- **API key management** with file permissions (0600)
- **Channel access control** via `allowFrom` allowlists
- **Command execution safety** with dangerous pattern blocking
- **Dependency security** with pip-audit and npm audit recommendations
- **Production deployment** best practices

**Known limitations documented:**
- No rate limiting
- Plain text config storage
- No session management
- Limited command filtering

---

## Release/Versioning Approach

### Version Format

Uses semantic versioning with post-releases: `v{MAJOR}.{MINOR}.{PATCH}.post{N}`

### Recent Releases

| Tag | Release Date |
|-----|--------------|
| v0.1.4.post5 | 2026-03-16 |
| v0.1.4.post4 | 2026-03-08 |
| v0.1.4.post3 | 2026-02-28 |
| v0.1.4.post2 | 2026-02-24 |
| v0.1.4.post1 | 2026-02-21 |
| v0.1.4 | 2026-02-21 |

**Release cadence:** Approximately weekly post-releases with bug fixes and features.

---

## CI/CD

### GitHub Actions Workflow

- **File:** [`.github/workflows/ci.yml`](.github/workflows/ci.yml)
- **Trigger:** Push to `main` and `nightly` branches; PRs to both branches
- **Python versions:** 3.11, 3.12, 3.13
- **Dependencies:** Uses `uv` for fast package management
- **System dependencies:** libolm-dev, build-essential
- **Test command:** `uv run pytest tests/`

---

## Community Channels

| Channel | Details |
|---------|---------|
| **Discord** | https://discord.gg/MnCvHqpUGB |
| **WeChat/Feishu** | QR codes available in [COMMUNICATION.md](./COMMUNICATION.md) |
| **Email** | xubinrencs@gmail.com (Xubin Ren / Re-bin) |

---

## Activity Indicators

### Commit Activity (2025-2026)

- **Total commits since 2025-01-01:** 1,468
- **Recent activity:** Multiple daily commits
- **Last 5 commits:**
  - 2026-03-26: fix telegram streaming message boundaries
  - 2026-03-25: feat(provider): add Step Fun provider support
  - 2026-03-25: feat(channel): add message send retry mechanism
  - 2026-03-25: feat(cron): inherit agent timezone for default schedules
  - 2026-03-25: fix(providers): add max_completion_tokens for OpenAI o1

### Repository Health Assessment

| Dimension | Status | Notes |
|-----------|--------|-------|
| **Maintainer engagement** | Active | Daily commits, responsive to issues |
| **Community growth** | High | 36k stars, 6k forks, 200+ contributors |
| **Release cadence** | Regular | Weekly post-releases |
| **CI/CD coverage** | Good | Multi-version Python testing |
| **Documentation** | Comprehensive | CONTRIBUTING, SECURITY, COMMUNICATION docs |
| **Issue response** | Active | 817 open issues, maintained |
| **Security practices** | Mature | Private reporting, documented best practices |

---

## Summary

nanobot is a **well-maintained, actively developed project** with:
- Strong community adoption (36k+ stars)
- Clear maintainership with two dedicated leads
- Comprehensive contributing guidelines emphasizing code quality
- Professional security practices
- Active development with regular releases
- Good CI/CD infrastructure
- Multiple community engagement channels

The project demonstrates healthy open-source practices with transparent governance, documented processes, and active community participation.
