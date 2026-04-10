# CI/CD Pipeline

## GitHub Actions Workflows

### Testing & Quality

| Workflow | Purpose |
|----------|---------|
| `python-package.yml` | Tests on Python 3.10, 3.11, 3.12 |
| `tests.yml` | flake8 lint + pytest |
| `code-quality-and-tests.yml` | Black + Ruff + full pytest with API keys |
| `test-main-features.yml` | Feature-specific tests |

### Security Scanning

| Workflow | Purpose |
|----------|---------|
| `codeql.yml` | GitHub CodeQL security analysis |
| `dependency-review.yml` | Dependency vulnerability scanning |
| `pysa.yml` | Meta's Python Static Analyzer |

### Code Quality

| Workflow | Purpose |
|----------|---------|
| `codacy.yml` | Codacy code quality analysis |
| `pyre.yml` | Meta's Pyre type checker |
| `lint.yml` | Black + Ruff linting |

### Documentation

| Workflow | Purpose |
|----------|---------|
| `docs.yml` | Documentation build |
| `docs-preview.yml` | Doc preview deployments |

### Automation & Bots

| Workflow | Purpose |
|----------|---------|
| `RELEASE.yml` | Release automation |
| `stale.yml` | Automated stale issue management |
| `welcome.yml` | New contributor welcome |
| `label.yml` | Automated issue/PR labeling |

## Testing Strategy

### Matrix Testing
Tests run on Python 3.10, 3.11, and 3.12 across all platforms.

### Dependency Caching
Poetry virtualenvs cached by lock file hash for faster builds.

### Secret Management
API keys (OPENAI_API_KEY, ANTHROPIC_API_KEY) managed via GitHub secrets.

## Linting Configuration

### Ruff
```toml
[tool.ruff]
line-length = 70
```

### Black
```toml
[tool.black]
target-version = ["py38"]
line-length = 70
```

### flake8 (tests.yml)
- Fails build on syntax errors or undefined names
- Treats complexity warnings as informational

## Release Process

### Trigger
- Merged PR to master with `release` label
- Changes to `pyproject.toml`

### Steps
1. Checkout code
2. Install Poetry 1.4.2
3. Set up Python 3.9
4. Build distribution with `poetry build`
5. Extract version from pyproject.toml
6. Create GitHub release with auto-generated release notes
7. Publish to PyPI using `POETRY_PYPI_TOKEN_PYPI` secret

## Docker Configuration

### Multi-Stage Build
| Stage | Base Image | Purpose |
|-------|------------|---------|
| builder | python:3.11-slim-bullseye | Install build deps, UV, create venv |
| final | python:3.11-slim-bullseye | Runtime image |

### Features
- **uv** package manager (Astral's fast Python package manager)
- Non-root `swarms` user for security
- Health check: `python -c "import swarms; print('Health check passed')"`
- Environment: PYTHONDONTWRITEBYTECODE=1, PYTHONUNBUFFERED=1
