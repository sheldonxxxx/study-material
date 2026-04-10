# OpenViking CI/CD

## Overview

OpenViking uses **GitHub Actions** for continuous integration and deployment. The project maintains **17 workflow files** with comprehensive coverage across build, test, security, and release processes.

## Workflow Architecture

### Automatic Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `pr.yml` | Pull request | PR validation (Lint + Lite tests) |
| `ci.yml` | Push to main | Main branch checks (Full tests + CodeQL) |
| `release.yml` | GitHub Release | PyPI + Docker publishing |
| `schedule.yml` | Weekly cron | Scheduled tasks |
| `rust-cli.yml` | Push to main, PR | Rust CLI multi-platform builds |
| `build-docker-image.yml` | Push | Docker image builds |

### Reusable Workflow Templates

| Template | Purpose |
|----------|---------|
| `_build.yml` | Python package builds |
| `_lint.yml` | Code linting (Ruff, Mypy) |
| `_test_full.yml` | Full test suite (all OS/Python versions) |
| `_test_lite.yml` | Quick test suite (Linux + Python 3.10) |
| `_codeql.yml` | Security scanning |
| `_publish.yml` | PyPI publishing |

## Build Pipeline

### Python Package Builds

**Build Platforms:**
- Linux x86_64 (glibc 2.31)
- Linux ARM64 (aarch64)
- macOS x86_64 (Intel)
- macOS ARM64 (Apple Silicon)
- Windows x86_64

**Python Versions:** 3.10, 3.11, 3.12, 3.13, 3.14

**Build Outputs:**
- Source distribution (sdist)
- Wheel packages (abi3 extensions)

### Rust CLI Builds

**Build Targets (5 platforms):**
- Linux x86_64 (glibc 2.31)
- Linux ARM64 (aarch64)
- macOS x86_64 (Intel)
- macOS ARM64 (Apple Silicon)
- Windows x86_64

## Test Strategy

### Lite Test Suite
- **Trigger**: Pull requests
- **Platform**: Linux only
- **Python Version**: 3.10
- **Purpose**: Fast feedback for contributors

### Full Test Suite
- **Trigger**: Push to main
- **Platforms**: Linux, macOS, Windows
- **Python Versions**: 3.10 - 3.14
- **Purpose**: Ensure main branch stability

## Release Process

### Version Management

- Uses `setuptools-scm` for automatic version extraction from Git tags
- **Tag format**: `vX.Y.Z` (e.g., `v0.2.13`)
- **Version scheme**: Semantic versioning

### Release Workflow

1. **Build Stage**
   - Builds sdist and wheels for all platforms
   - Runs full test suite
   - Generates Docker images

2. **Validation Stage**
   - Permission checks (admin/maintain/write required)

3. **Publish Stage**
   - Publish to TestPyPI first (recommended)
   - Publish to PyPI on success
   - Skip overwrites (version must be unique)

4. **Docker Publishing**
   - Multi-platform images (linux/amd64, linux/arm64)
   - Published to ghcr.io
   - Automatic tag/labels from release metadata

### Release Targets

| Target | Purpose |
|--------|---------|
| `none` | Build artifacts only (no publish) |
| `testpypi` | Beta testing on TestPyPI |
| `pypi` | Official PyPI release |
| `both` | Publish to both repositories |

## Security

### CodeQL Analysis

- **Automatic**: On push to main
- **Scheduled**: Weekly (Sunday)
- **Scope**: Full codebase security scanning

### Dependency Management

- **Dependabot**: Enabled (`dependabot.yml`)
- Automatically updates dependencies

## CI/CD Pipeline Diagram

```
PR Created
    │
    ▼
┌─────────────────┐
│   pr.yml        │
│  - Lint         │
│  - Test Lite    │
└────────┬────────┘
         │
         ▼ (merge to main)
┌─────────────────┐
│   ci.yml        │
│  - Test Full    │
│  - CodeQL       │
└────────┬────────┘
         │
         ▼ (create release)
┌─────────────────┐
│  release.yml    │
│  - Build        │
│  - Test         │
│  - Publish PyPI │
│  - Docker       │
└─────────────────┘
```

## Development Integration

### Pre-commit Hooks

The project uses `pre-commit` for local CI:

```bash
# Install
pip install pre-commit
pre-commit install

# Checks run automatically on commit:
# - ruff (lint + format)
# - mypy (type check)
```

### Manual Workflow Triggers

Maintainers can manually trigger workflows from the GitHub Actions tab:

| Workflow | Purpose |
|----------|---------|
| Lint Checks | Code style verification |
| Test Suite (Lite) | Fast integration tests |
| Test Suite (Full) | Complete test matrix |
| CodeQL Scan | Security analysis |
| Build Distribution | Build packages only |
| Publish Distribution | Publish to PyPI |
| Manual Release | One-stop build and publish |

## Docker

### Multi-Stage Dockerfile

1. **go-toolchain**: Go 1.26 for AGFS server builds
2. **rust-toolchain**: Rust 1.88 for CLI builds
3. **py-builder**: Python 3.13 with uv, builds all artifacts
4. **runtime**: Python 3.13 slim runtime image

### Docker Compose

Single service `openviking` exposing port 1933 with health checks.
