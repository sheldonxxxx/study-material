# Tech Stack Analysis: CAMEL-Oasis

## Project Overview

**Name:** camel-oasis
**Version:** 0.2.5
**Description:** Open Agents Social Interaction Simulations on a Large Scale
**Repository:** https://github.com/camel-ai/oasis.git
**License:** Apache 2.0

---

## Language and Runtime

| Attribute | Value |
|-----------|-------|
| Language | Python |
| Version Constraint | `>=3.10.0, <3.12` |
| Target in CI | Python 3.10.11 |
| Container Base | `python:3.10-bookworm` |

---

## Dependency Management

### Tool
- **Poetry** with `poetry-core>=1.0.0` as build backend
- Lock file: `poetry.lock` (374KB)

### Key Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| camel-ai | 0.2.78 | Multi-agent framework (core dependency) |
| pandas | 2.2.2 | Data manipulation |
| sentence-transformers | 3.0.0 | Sentence embeddings |
| neo4j | 5.23.0 | Graph database driver |
| igraph | 0.11.6 | Graph algorithms |
| cairocffi | 1.7.1 | Cairo graphics bindings |
| pillow | 10.3.0 | Image processing |
| unstructured | 0.13.7 | Document parsing |
| slack_sdk | 3.31.0 | Slack API integration |
| requests_oauthlib | 2.0.0 | OAuth support |
| pytest | 8.2.0 | Testing |
| pytest-asyncio | 0.23.6 | Async test support |
| pre-commit | 3.7.1 | Git hooks |

---

## Build and Packaging

### Build System
- **Poetry** (pyproject.toml with `[tool.poetry]` section)
- **uv** used in Docker container for fast pip installs

### Docker Support
- Location: `.container/Dockerfile`
- Base image: `python:3.10-bookworm`
- Features:
  - Uses `uv` for fast package installation
  - Installs dev dependencies via `poetry install -e ".[dev]"`
  - Pre-commit hooks installed
  - Dev container with live-tail for keepalive

### Deployment
- `deploy.py` at root (custom deployment script)

---

## CI/CD

### GitHub Actions
- **Workflow:** `.github/workflows/python-app.yml`
- **Triggers:** Push to main, pull requests to main
- **Runner:** ubuntu-latest
- **Steps:**
  1. Checkout code
  2. Set up Python 3.10.11
  3. Install dependencies (`pip install -e .`)
  4. Install and run pre-commit hooks
  5. Run pytest (requires `OPENAI_BASE_URL` and `OPENAI_API_KEY` secrets)

---

## Code Quality and Linting

### Pre-commit Hooks
Configured in `.pre-commit-config.yaml`:

| Hook | Version | Purpose |
|------|---------|---------|
| end-of-file-fixer | v4.5.0 | Ensure files end with newline |
| mixed-line-ending | v4.5.0 | Normalize to LF |
| trailing-whitespace | v4.5.0 | Remove trailing whitespace |
| check-toml | v4.5.0 | Validate TOML |
| check-yaml | v4.5.0 | Validate YAML |
| yamllint | v1.35.1 | Lint YAML files |
| isort | 5.13.2 | Sort Python imports |
| flake8 | 7.0.0 | PEP8 linting |
| ruff | v0.3.5 | Fast formatting/linting |
| mdformat | v0.7.17 | Format Markdown |
| check-license | local | Verify license headers |

---

## Monorepo and Project Structure

### Structure
```
oasis/
  __init__.py       # Main package init
  clock/            # Time simulation
  environment/      # Environment simulation
  social_agent/     # Agent implementations
  social_platform/  # Platform integrations
  testing/          # Test utilities

examples/           # Example scripts
docs/               # Documentation
test/               # Test suite
visualization/      # Visualization tools
generator/          # Data generators
assets/             # Static assets
data/                # Data files
```

### Monorepo Tooling
**None detected.** Single Python package managed by Poetry. No Nx, Turborepo, Lerna, or pnpm-workspace configuration.

---

## Summary

| Aspect | Choice |
|--------|--------|
| Language | Python 3.10-3.11 |
| Package Manager | Poetry |
| Build Backend | poetry-core |
| Container | Docker (python:3.10-bookworm) |
| CI/CD | GitHub Actions |
| Code Quality | Ruff, Flake8, isort, yamllint, mdformat |
| Testing | pytest, pytest-asyncio |
| Monorepo | No (single package) |
| Key Framework | camel-ai (multi-agent) |
| Graph DB | Neo4j |
| Embeddings | sentence-transformers |
