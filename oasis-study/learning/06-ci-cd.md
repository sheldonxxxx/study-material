# CI/CD Pipeline

## Overview

OASIS uses GitHub Actions for continuous integration with a focus on code quality gates and automated testing.

## Pipeline Configuration

### Trigger Conditions
- Push to `main` branch
- Pull requests targeting `main` branch

### Build Environment
- **Runner**: Ubuntu Latest
- **Python Version**: 3.10.11

## Pipeline Stages

### 1. Checkout
Standard GitHub Actions checkout step using `actions/checkout@v2`

### 2. Python Setup
```yaml
- uses: actions/setup-python@v2
  with:
    python-version: 3.10.11
```

### 3. Dependency Installation
```bash
python -m pip install --upgrade pip
pip install -e .
```

### 4. Pre-commit Installation & Execution
```bash
pip install pre-commit
pre-commit run --all-files
```

### 5. Test Execution
Requires GitHub Secrets:
- `OPENAI_BASE_URL` - API endpoint
- `OPENAI_API_KEY` - Authentication key

```bash
pytest
```

## Pre-commit Hooks

OASIS enforces code quality through pre-commit hooks configured in `.pre-commit-config.yaml`:

| Hook | Purpose |
|------|---------|
| end-of-file-fixer | Ensures files end with newline |
| mixed-line-ending | Normalizes line endings to LF |
| trailing-whitespace | Removes trailing whitespace |
| check-toml | Validates TOML files |
| check-yaml | Validates YAML files |
| yamllint | Lints YAML with custom rules |
| isort | Sorts Python imports |
| flake8 | PEP 8 compliance checking |
| ruff | Code formatting and linting |
| mdformat | Markdown formatting |
| check-license | Validates Apache 2.0 license headers |

## Local Development Checks

Contributors should run these commands before pushing:

```bash
# Install dependencies
poetry install

# Activate virtual environment
eval $(poetry env activate)

# Install pre-commit hooks
pre-commit install

# Run all checks
pre-commit run --all-files

# Run tests
pytest test
```

## Deployment

Documentation is automatically deployed via Mintlify GitHub App when changes are pushed to main branch, updating https://docs.oasis.camel-ai.org/

## Gaps & Observations

1. **No explicit deployment job** - Only CI, no CD for the PyPI package
2. **Tests require API keys** - Cannot run full test suite without OpenAI credentials
3. **Single Python version** - Only tests against 3.10.11, not multiple versions
4. **No Docker/CI for containerized deployment** - Though `.container/` directory exists
5. **No security scanning** - No automated dependency vulnerability checks
6. **No coverage reporting** - Coverage is documented for local use but not in CI
