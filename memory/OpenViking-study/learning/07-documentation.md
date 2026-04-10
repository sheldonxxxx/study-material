# OpenViking Documentation

## Overview

OpenViking maintains comprehensive documentation with **multi-language support** (English, Chinese, Japanese), structured contributing guidelines, and well-organized API documentation.

## README Quality

### Strengths

- **Multi-language**: Available in English (`README.md`), Chinese (`README_CN.md`), and Japanese (`README_JA.md`)
- **Visual Design**: Professional banner logo, shields for stats, star history chart
- **Community Links**: Lark Group, WeChat, Discord, X (Twitter)
- **Quick Start**: Complete installation and configuration walkthrough
- **Provider Support**: Detailed examples for Volcengine, OpenAI, and LiteLLM (Claude, DeepSeek, Gemini, Qwen, vLLM, Ollama)
- **Architecture Explanation**: Core concepts with visual directory tree examples
- **Performance Metrics**: OpenClaw integration benchmarks showing 43-49% improvement

### Coverage

| Section | Status |
|---------|--------|
| Overview/Challenges | Complete |
| Solution/Five Core Concepts | Complete |
| Quick Start (Installation, Config, Run) | Complete |
| Server Deployment Guide | Complete |
| Advanced Reading Links | Complete |
| Community/Join | Complete |
| License | Complete |

### README Statistics

- **Lines**: ~675 lines
- **Sections**: 10+ major sections
- **Code Examples**: 20+ configuration examples
- **Provider Guides**: 3 (Volcengine, OpenAI, LiteLLM)

## Documentation Structure

```
docs/
├── images/                     # Visual assets
├── en/                        # English documentation
│   ├── about/
│   │   ├── 01-about-us.md
│   │   ├── 02-changelog.md
│   │   └── 03-roadmap.md
│   ├── api/
│   │   ├── 01-overview.md
│   │   ├── 02-resources.md
│   │   ├── 03-filesystem.md
│   │   ├── 04-skills.md
│   │   ├── 05-sessions.md
│   │   ├── 06-retrieval.md
│   │   ├── 07-system.md
│   │   └── 08-admin.md
│   ├── concepts/
│   │   ├── 01-architecture.md
│   │   ├── 02-context-types.md
│   │   ├── 03-context-layers.md
│   │   ├── 04-viking-uri.md
│   │   ├── 05-storage.md
│   │   ├── 06-extraction.md
│   │   ├── 07-retrieval.md
│   │   ├── 08-session.md
│   │   ├── 09-transaction.md
│   │   └── 10-encryption.md
│   ├── faq/
│   │   └── faq.md
│   ├── getting-started/
│   │   ├── 01-introduction.md
│   │   ├── 02-quickstart.md
│   │   └── 03-quickstart-server.md
│   └── guides/
│       ├── 01-configuration.md
│       ├── 02-volcengine-purchase-guide.md
│       ├── 03-deployment.md
│       ├── 04-authentication.md
│       ├── 05-monitoring.md
│       ├── 06-mcp-integration.md
│       ├── 07-operation-telemetry.md
│       └── 08-encryption.md
└── zh/                        # Chinese documentation (parallel structure)
    ├── about/
    ├── api/
    ├── concepts/
    ├── faq/
    ├── getting-started/
    └── guides/
```

### Documentation Categories

| Category | Files | Purpose |
|----------|-------|---------|
| Getting Started | 3 | Introduction, quickstart, server setup |
| Concepts | 10 | Architecture, context types, layers, storage, retrieval |
| API Reference | 8 | Resources, filesystem, skills, sessions, retrieval, system, admin |
| Guides | 8 | Configuration, deployment, authentication, monitoring, encryption |
| About | 3 | About us, changelog, roadmap |
| FAQ | 1 | Frequently asked questions |

## CONTRIBUTING Guidelines

### Multi-language Support

- `CONTRIBUTING.md` (English)
- `CONTRIBUTING_CN.md` (Chinese)
- `CONTRIBUTING_JA.md` (Japanese)

### Prerequisites Documentation

| Requirement | Version | Purpose |
|-------------|---------|---------|
| Python | 3.10+ | Primary development |
| Go | 1.22+ | AGFS components |
| Rust | 1.88+ | CLI builds |
| C++ Compiler | GCC 9+ / Clang 11+ | Core extensions |
| CMake | 3.12+ | Build system |

### Development Setup

1. **Fork and Clone**
2. **Install Dependencies**: `uv sync --all-extras`
3. **Configure Environment**: `~/.openviking/ov.conf`
4. **Verify Installation**: Run sample code
5. **Build Rust CLI** (optional)

### Code Style

| Tool | Purpose | Config |
|------|---------|--------|
| Ruff | Linting, formatting, import sorting | `pyproject.toml` |
| Mypy | Type checking | `pyproject.toml` |

**Guidelines:**
- Line width: 100 characters
- Indentation: 4 spaces
- Strings: Double quotes
- Type hints: Encouraged
- Docstrings: Required for public APIs

### Testing

- **Framework**: pytest with asyncio support
- **Fixtures**: Common fixtures in `tests/conftest.py`
- **Coverage**: `--cov=openviking` support

### Commit Convention

Follows **Conventional Commits**:

```
<type>(<scope>): <subject>
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`

## Issue Templates

| Template | Purpose | Labels |
|----------|---------|--------|
| Bug Report | Bugs with reproduction steps | bug |
| Feature Request | New features with use cases | enhancement |
| Question | Usage questions | question |

### Issue Configuration

- Blank issues disabled
- Links to documentation, Lark community, GitHub Discussions

## Pull Request Template

**Required Sections:**
- Summary description
- Type of change checklist
- Testing description
- Related issues
- Checklist (code style, tests, docs, warnings)

## Documentation Metrics

| Metric | Value |
|--------|-------|
| Total Doc Pages | ~60+ |
| Languages | 3 (EN, ZH, JA) |
| API Endpoints Documented | 8 categories |
| Core Concepts | 10 topics |
| Getting Started Guides | 3 |
| Deployment Guides | 8 |

## Areas for Improvement

1. **No CODEOWNERS file** - unclear code ownership
2. **No MAINTAINERS file** - no explicit maintainer list
3. **No AUTHORS file** - contributor recognition gap
4. **No public ROADMAP.md** in repo (exists in docs/)
5. **No security policy** (security.md not enabled)

## Quick Reference Links

| Resource | URL |
|----------|-----|
| Documentation | https://www.openviking.ai/docs |
| Website | https://www.openviking.ai |
| GitHub Issues | https://github.com/volcengine/OpenViking/issues |
| Discussions | https://github.com/volcengine/OpenViking/discussions |
