# Tech Stack & Choices Analysis: Swarms

## Overview

**Project:** swarms (Swarms - TGSC)
**Version:** 10.0.1
**Repository:** https://github.com/kyegomez/swarms
**License:** MIT

---

## Python Version Requirements

| Setting | Value |
|---------|-------|
| Required Version | `>=3.10,<4.0` |
| Docker Base Image | `python:3.11-slim-bullseye` |
| CI Test Matrix | Python 3.10, 3.11, 3.12 |

**Analysis:** The project targets Python 3.10+ but Docker deployments use 3.11 for stability. The upper bound of `<4.0` indicates active maintenance but no rush to adopt Python 4.x.

---

## Build System Configuration

| Component | Choice |
|-----------|--------|
| Build Backend | `poetry-core>=1.0.0` |
| Package Manager | Poetry |
| Config File | `pyproject.toml` |

The project uses **Poetry** as the primary package manager and build tool, with `pyproject.toml` as the single source of truth for all dependency and tooling configuration.

### Build Backend Details

```toml
[build-system]
requires = ["poetry-core>=1.0.0"]
build-backend = "poetry.core.masonry.api"
```

---

## Key Dependencies & Their Purposes

### Core Runtime Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| `asyncio` | `>=3.4.3,<5.0` | Async/await concurrency |
| `aiofiles` | `*` | Async file I/O |
| `aiohttp` | `*` | Async HTTP client/server |
| `httpx` | `*` | Sync/async HTTP client |
| `uvloop` | `*` (linux/darwin) | Fast asyncio event loop |
| `winloop` | `*` (windows) | Fast asyncio event loop for Windows |

### AI/LLM Integration

| Dependency | Version | Purpose |
|------------|---------|---------|
| `litellm` | `1.76.1` | Unified LLM API interface |
| `pydantic` | `*` | Data validation and settings |
| `mcp` | `*` | Model Context Protocol |

### Data Processing & Utilities

| Dependency | Version | Purpose |
|------------|---------|---------|
| `pypdf` | `5.1.0` | PDF text extraction |
| `numpy` | `*` | Numerical computing |
| `networkx` | `*` | Graph data structures |
| `psutil` | `*` | System/resource monitoring |

### Logging & Output

| Dependency | Version | Purpose |
|------------|---------|---------|
| `loguru` | `*` | Structured logging |
| `rich` | `*` | Terminal formatting/colors |

### Reliability Patterns

| Dependency | Version | Purpose |
|------------|---------|---------|
| `tenacity` | `*` | Retry logic |
| `ratelimit` | `2.2.1` | API rate limiting |

### Configuration & Environment

| Dependency | Version | Purpose |
|------------|---------|---------|
| `python-dotenv` | `*` | .env file loading |
| `PyYAML` | `*` | YAML config parsing |
| `toml` | `*` | TOML parsing |

### CLI & Scheduling

| Dependency | Version | Purpose |
|------------|---------|---------|
| `schedule` | `*` | Job scheduling |

---

## Linting, Type Checking & Testing

### Development Dependencies

| Group | Dependencies |
|-------|--------------|
| **lint** | black, ruff, types-toml, types-pytz, types-chardet, mypy-protobuf |
| **test** | pytest >=8.1.1,<10.0.0 |
| **dev** | black, ruff, pytest |

### Code Quality Tools Configuration

**Ruff** (line-length: 70):
```toml
[tool.ruff]
line-length = 70
```

**Black** (target-version: py38, line-length: 70):
```toml
[tool.black]
target-version = ["py38"]
line-length = 70
```

---

## CI/CD Configuration

### GitHub Actions Workflows

| Workflow | Purpose |
|----------|---------|
| `python-package.yml` | Runs tests on Python 3.10, 3.11, 3.12 |
| `tests.yml` | Basic flake8 lint + pytest |
| `code-quality-and-tests.yml` | Black + Ruff + full pytest with API keys |
| `codacy.yml` | Code quality analysis |
| `codeql.yml` | GitHub security analysis |
| `dependency-review.yml` | Dependency vulnerability scanning |
| `docs.yml` | Documentation build |
| `docs-preview.yml` | Doc preview deployments |
| `pyre.yml` | Meta's type checker |
| `pysa.yml` | Meta's static analyzer |
| `RELEASE.yml` | Release automation |
| `test-main-features.yml` | Feature-specific tests |
| `label.yml`, `stale.yml`, `welcome.yml` | Bot/management workflows |

### CI Testing Strategy

- **Matrix Testing:** Tests run on Python 3.10, 3.11, 3.12
- **Dependency Caching:** Poetry virtualenvs cached by lock file hash
- **Secret Management:** API keys (OPENAI_API_KEY, ANTHROPIC_API_KEY) via GitHub secrets
- **Artifact Upload:** Test results and .coverage files uploaded post-run

---

## Docker Configuration

### Multi-Stage Dockerfile

| Stage | Base Image | Purpose |
|-------|------------|---------|
| **builder** | `python:3.11-slim-bullseye` | Install build deps, UV, create venv |
| **final** | `python:3.11-slim-bullseye` | Runtime image |

### Key Docker Features

- **Package Manager:** Uses `uv` (Astral's fast Python package manager) instead of pip
- **Non-root User:** Creates `swarms` user for security
- **Health Check:** `python -c "import swarms; print('Health check passed')"`
- **Environment:** PYTHONDONTWRITEBYTECODE=1, PYTHONUNBUFFERED=1

---

## Project Structure

```
swarms/
├── __init__.py          # Main exports + telemetry bootup
├── env.py               # Environment loading
├── agents/              # Agent implementations (46 files)
├── artifacts/          # Output/artifacts handling
├── cli/                # Command-line interface
├── prompts/            # Prompt templates (69 files)
├── schemas/            # Data schemas/models (13 files)
├── sims/               # Simulation modules
├── structs/            # Data structures (65 files)
├── telemetry/          # Bootup and telemetry
├── tools/              # Tool integrations (20 files)
└── utils/              # Utility functions (29 files)
```

### Module Breakdown

- **agents/** - Multi-agent system implementations (council, debate, hierarchical, router, etc.)
- **structs/** - Core data structures (Agent, Task, Workflow, etc.)
- **prompts/** - Pre-built prompt templates for various agent types
- **tools/** - External tool integrations
- **utils/** - Helper utilities

---

## Summary of Architectural Choices

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| **Package Manager** | Poetry | Modern, lockfile-based, good for libs |
| **Python Constraint** | 3.10 - 4.0 | Modern async syntax, broad compatibility |
| **Async Runtime** | asyncio + uvloop | Performance on Linux/Mac, Windows fallback |
| **LLM Abstraction** | litellm | Unified API for 50+ LLM providers |
| **Data Validation** | pydantic | Type safety, settings management |
| **Graph Structures** | networkx | Agent interaction patterns |
| **Logging** | loguru | Simpler than logging module |
| **CLI Output** | rich | Pretty terminal output |
| **PDF Processing** | pypdf | Lightweight PDF text extraction |
| **Type Checking** | black + ruff + mypy-protobuf | Fast linting + type safety |

---

## Notes

- The project has extensive CI/CD coverage including security scanning (CodeQL, dependency-review), code quality (Codacy, Ruff, Black), and type checking (Pyre, mypy-protobuf)
- The `mcp` dependency suggests Model Context Protocol integration for extended capabilities
- Telemetry bootup on import indicates some form of analytics or monitoring
- The CLI script entry point (`swarms.cli.main:main`) indicates a command-line interface exists
