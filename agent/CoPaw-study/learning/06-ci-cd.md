# CoPaw CI/CD Pipeline

## Overview

CoPaw uses **GitHub Actions** for CI/CD with comprehensive testing, multi-platform builds, and automated releases to multiple registries.

## Workflows

### 1. Tests (`tests.yml`)

**Trigger:** Push/PR to main/master/dev/develop branches; workflow_dispatch

**Matrix:**
- Python: 3.10, 3.13
- OS: Ubuntu, macOS, Windows

**Jobs:**
1. **approval-gate** - Maintainer approval required before CI runs
2. **unit-tests** - Runs `pytest tests/unit -v`
3. **integrated-tests** - Runs `pytest tests/integrated -v` (if exist)
4. **coverage-report** - Generates coverage with PR comments via orgoro/coverage
5. **test-summary** - Aggregates all test results

**Key Features:**
- Console frontend is built before tests run (`cd console && npm ci && npm run build`)
- Build output copied to `src/copaw/console/` for testing
- Coverage reports posted as PR comments
- macOS uses special handling for MLX and llama-cpp (prebuilt wheels)

---

### 2. Docker Release (`docker-release.yml`)

**Trigger:** Release published; workflow_dispatch

**Registry Targets:**
- Docker Hub: `agentscope/copaw`
- Aliyun ACR: `agentscope-registry.ap-southeast-1.cr.aliyuncs.com/agentscope/copaw`

**Tags:**
- `${VERSION}` and `pre` - Always pushed
- `latest` - Only for stable releases (not beta/alpha/rc/dev)

**Build:**
- Multi-arch: `linux/amd64, linux/arm64`
- Platform: Ubuntu-based with Python 3, Chromium, XFCE
- Argument: `COPAW_DISABLED_CHANNELS=imessage` (iMessage not available in Docker)

---

### 3. Desktop Release (`desktop-release.yml`)

**Trigger:** Release published

**Platforms:**
- Windows: `CoPaw-Setup-${version}.exe`
- macOS: `CoPaw-${version}-macOS.zip` (Apple Silicon recommended)

---

### 4. Publish to PyPI (`publish-pypi.yml`)

**Trigger:** Release published

**Action:** Builds and publishes Python package to PyPI

---

### 5. Deploy Website (`deploy-website.yml`)

**Trigger:** Push to main branch

**Action:** Deploys documentation site

---

### 6. Pre-commit (`pre-commit.yml`)

**Trigger:** Pull requests

**Purpose:** Runs pre-commit hooks on PR changes

---

### 7. npm Format (`npm-format.yml`)

**Trigger:** Pull requests

**Purpose:** Formats console and website code

---

### 8. Welcome Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `issue-welcome.yml` | Issue opened | Automated issue responses |
| `pr-welcome.yml` | PR opened | Automated PR responses |
| `first-time-contributor-welcome.yml` | First-time contributor PR | Onboarding assistance |

---

## Testing Strategy

### Test Structure
```
tests/
├── unit/           # Unit tests
└── integrated/     # Integration tests (optional)
```

### Test Configuration (pyproject.toml)
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "function"
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
]
```

### Coverage
- Minimum threshold: 0.0 (no强制要求)
- Coverage comment posted on PRs
- Reports: XML, term-missing, HTML

---

## Local Development Gate

Before pushing or opening a PR, run:

```bash
# Install with dev + full dependencies
pip install -e ".[dev,full]"

# Install pre-commit hooks
pre-commit install

# Run pre-commit checks
pre-commit run --all-files

# Run all tests
pytest

# Frontend formatting (if changing console/website)
cd console && npm run format
cd website && npm run format
```

---

## Release Process

### Version Bumping
- Versions managed via `src/copaw/__version__.__version__` (dynamic)
- Semantic versioning: `v0.0.x` through `v0.2.0`
- Beta pre-releases: `v0.1.0-beta.1` through `v0.1.0-beta.4`
- Post-releases: `.post1` suffix for quick patches

### Release Checklist
1. Update version in source
2. Create GitHub release (triggers all release workflows)
3. Docker images built and pushed to Docker Hub + Aliyun ACR
4. Desktop builds created
5. PyPI package published
6. Website documentation deployed

---

## Docker Image Architecture

### Base Image Layers
1. **Build stage**: Node.js slim for frontend build
2. **Runtime stage**: Debian-based with Python 3

### Runtime Includes
- Python 3.x
- Chromium browser (for playwright)
- XFCE (for headless GUI)
- Supervisor (process management)
- Xvfb (virtual framebuffer)

### Volume Mounts
- `copaw-data:/app/working` - Config, memory, skills
- `copaw-secrets:/app/working.secret` - API keys, secrets

### Environment Variables
- `DASHSCOPE_API_KEY` - DashScope API key
- `COPAW_DISABLED_CHANNELS` - Channels to disable (e.g., `imessage`)

---

## Secrets Required

| Secret | Purpose |
|--------|---------|
| `DOCKER_USERNAME` | Docker Hub login |
| `DOCKER_PASSWORD` | Docker Hub password |
| `ALIYUN_ACR_USERNAME` | Aliyun ACR login |
| `ALIYUN_ACR_PASSWORD` | Aliyun ACR password |
| `MAINTAINER_TOKEN` | For approval gate |

---

## CI/CD Quality Gates

1. **Pre-commit** must pass on all PRs
2. **Maintainer approval** required before tests run
3. **All unit tests** must pass across Python 3.10, 3.12, 3.13
4. **All integration tests** must pass (if exist)
5. **Coverage** generated but not enforced
6. **npm format** must pass for frontend changes
