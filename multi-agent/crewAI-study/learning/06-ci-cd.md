# CI/CD Pipeline

## Overview

CrewAI uses GitHub Actions for continuous integration and deployment. The repository has comprehensive automation covering testing, linting, type checking, security scanning, and releases.

## Workflow Files

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `tests.yml` | Every PR | Run test suites across Python versions |
| `type-checker.yml` | Every PR | mypy type checking on Python 3.10-3.13 |
| `linter.yml` | Every PR | ruff check and format validation |
| `pr-title.yml` | PR open/edit | Validate conventional commit format |
| `codeql.yml` | Push to main, every PR | Security vulnerability scanning |
| `publish.yml` | Manual workflow dispatch | Release to PyPI |
| `nightly.yml` | Daily at 6am UTC | Canary releases with nightly versioning |
| `pr-size.yml` | Every PR | Label PRs by size (auto-generated) |
| `stale.yml` | Daily | Mark stale issues/PRs |
| `docs-broken-links.yml` | Manual/periodic | Validate documentation links |
| `generate-tool-specs.yml` | Manual | Generate tool specifications |
| `update-test-durations.yml` | Periodic | Update test duration cache |
| `build-uv-cache.yml` | Manual | Build and cache uv environment |

## Test Pipeline

### Matrix Strategy

```yaml
strategy:
  matrix:
    python-version: ['3.10', '3.11', '3.12', '3.13']
    group: [1, 2, 3, 4, 5, 6, 7, 8]
```

Tests run across 4 Python versions x 8 parallel groups = 32 parallel test jobs.

### Test Splitting

- Uses `pytest-split` to divide tests evenly across 8 groups
- Duration-based test splitting (with fallback to even splitting)
- `--maxfail=3` to fail fast on repeated failures
- `pytest-xdist` for parallel execution within groups

### Test Commands

```bash
# crewai tests
cd lib/crewai && uv run pytest -vv --splits 8 --group N --durations=10 --maxfail=3

# crewai-tools tests
cd lib/crewai-tools && uv run pytest -vv --splits 8 --group N --durations=10 --maxfail=3
```

### Caching Strategy

1. **uv cache** - Restored/saved based on `uv.lock` hash + Python version
2. **test durations** - Cached per Python version for optimal splitting

```yaml
- name: Restore global uv cache
  uses: actions/cache/restore@v4
  with:
    path: |
      ~/.cache/uv
      ~/.local/share/uv
      .venv
    key: uv-main-py${{ matrix.python-version }}-${{ hashFiles('uv.lock') }}
```

## Type Checking

### Matrix

Runs mypy on all 4 supported Python versions:

```yaml
strategy:
  matrix:
    python-version: ["3.10", "3.11", "3.12", "3.13"]
```

### Configuration

- Strict mode enabled
- Runs on `lib/` directory
- Summary job aggregates all matrix results for branch protection

### Validation

```bash
uv run mypy lib/
```

## Linting

### Ruff Checks

```bash
# Code style and rules
uv run ruff check lib/

# Format validation
uv run ruff format --check lib/
```

### PR Title Validation

Enforces conventional commit format via `amannn/action-semantic-pull-request`:

```yaml
with:
  types: |
    feat fix refactor perf test docs chore ci style revert
  requireScope: false
  subjectPattern: ^[a-z].+[^.]$
```

## Security Scanning

### CodeQL

```yaml
on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]
```

Analyzes:
- `actions` (GitHub Actions workflow files)
- `python` (Python source code)

Build mode: none (interpreted language).

### Bandit Security Linting

Included in ruff's `S` rule set for security issues.

## Publishing Pipeline

### Release Trigger

Manual workflow dispatch with optional release tag:

```yaml
on:
  workflow_dispatch:
    inputs:
      release_tag:
        description: 'Release tag to publish'
        required: false
        type: string
```

### Build Process

```bash
uv build --all-packages
rm dist/.gitignore
```

Builds all workspace packages (crewai, crewai-tools, crewai-files).

### PyPI Publication

```yaml
permissions:
  id-token: write  # OIDC for trusted publishing
env:
  UV_PUBLISH_TOKEN: ${{ secrets.PYPI_API_TOKEN }}
```

Uses uv for publication with OIDC token authentication (no password).

### Post-Release Notifications

- Parses release notes from GitHub
- Posts Slack notification with version, PyPI link, and changelog summary

## Nightly Canary Releases

### Schedule

```yaml
on:
  schedule:
    - cron: '0 6 * * *'  # Daily at 6am UTC
```

### Version Stamping

```bash
DATE=$(date +%Y%m%d)
NIGHTLY="${CURRENT}.dev${DATE}"
# Updates __version__ in __init__.py files
# Updates cross-package dependency pins
```

Packages get version like `1.13.0.dev20260327`.

### Conditional Execution

```yaml
needs: check
if: needs.check.outputs.has_changes == 'true' || github.event_name == 'workflow_dispatch'
```

Only publishes if there are new commits in the last 24 hours.

## Pre-commit Hooks

### Local Hooks

```yaml
- id: ruff
- id: ruff-format
- id: mypy
```

### Remote Hooks

- `astral-sh/uv-pre-commit` - Ensures `uv.lock` is up to date
- `commitizen-tools/commitizen` - Validates commit messages

## Dependency Management

### Version Constraints

Critical overrides in `pyproject.toml`:

```toml
[tool.uv]
override-dependencies = [
    "rich>=13.7.1",
    "onnxruntime<1.24; python_version < '3.11'",
    "pillow>=12.1.1",
    "langchain-core>=1.2.11,<2",  # CVE-2026-26013 fix
    "urllib3>=2.6.3",
]
```

### Lock File

`uv.lock` is committed and validated by pre-commit.

## Cache Strategy Summary

| Cache Key | Restore Keys | Purpose |
|-----------|--------------|---------|
| `uv-main-py{python-version}-{lock-hash}` | `uv-main-py{python-version}-` | uv artifacts + venv |
| `test-durations-py{python-version}` | (none) | pytest duration data |

## Quality Gates

All PRs must pass before merge:
1. Ruff linting + formatting
2. mypy type checking (4 Python versions)
3. pytest on crewai + crewai-tools (32 parallel jobs)
4. PR title conventional commit validation
5. CodeQL security scan

## Artifacts

- `dist/` - Published to PyPI on release
- `.test_durations_py*` - Test duration cache files
