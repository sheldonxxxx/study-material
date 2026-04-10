# CI/CD

## Overview

DeerFlow uses GitHub Actions for continuous integration and delivery. The project maintains two primary workflows that run on every push and pull request to ensure code quality and prevent regressions.

## Workflows

### 1. Lint Check (`lint-check.yml`)

**Trigger**: Push to `main` branch, all pull requests

**Jobs**:

#### Backend Lint Job
```
- Checkout code
- Set up Python 3.12
- Install uv (Astral's fast Python package manager)
- Install backend dependencies: uv sync --group dev
- Run: make lint (ruff linter)
```

#### Frontend Lint Job
```
- Checkout code
- Set up Node.js 22
- Enable Corepack
- Use pinned pnpm version (10.26.2)
- Install dependencies: pnpm install --frozen-lockfile
- Run: pnpm lint (ESLint)
- Run: pnpm typecheck (TypeScript)
- Run: pnpm build (with BETTER_AUTH_SECRET=local-dev-secret)
```

### 2. Unit Tests (`backend-unit-tests.yml`)

**Trigger**: Push to `main` branch, pull requests (opened, synchronize, reopened, ready_for_review)

**Concurrency**: Cancels in-progress runs for the same PR/branch

**Conditions**: Skipped for draft PRs

**Timeout**: 15 minutes

**Steps**:
```
- Checkout code (v6)
- Set up Python 3.12
- Install uv
- Install dependencies: uv sync --group dev
- Run: make test (pytest)
```

## Regression Tests

DeerFlow maintains focused regression coverage for critical paths:

### Docker Sandbox Mode Detection
- **File**: `tests/test_docker_sandbox_mode_detection.py`
- **Purpose**: Verifies correct sandbox mode detection from `config.yaml`
- **Coverage**: Ensures Docker provisioner mode is correctly identified

### Provisioner Kubeconfig Handling
- **File**: `tests/test_provisioner_kubeconfig.py`
- **Purpose**: Tests kubeconfig file and directory handling for Kubernetes sandbox
- **Coverage**: Validates path resolution for kubeconfig contexts

### Harness Boundary Enforcement
- **File**: `tests/test_harness_boundary.py`
- **Purpose**: Ensures `packages/harness/deerflow/` never imports from `app.*`
- **Coverage**: Enforces architectural separation between harness and application layers

### Gateway Conformance Tests
- **File**: `tests/test_client.py` - `TestGatewayConformance`
- **Purpose**: Validates embedded client methods return Gateway-aligned response schemas
- **Coverage**: All dict-returning `DeerFlowClient` methods validated against Pydantic models

## Quality Gates

| Check | Backend | Frontend |
|-------|---------|----------|
| Linting | ruff | ESLint |
| Type Checking | - | TypeScript |
| Tests | pytest | - |
| Build | - | Next.js build |

## Environment

- **Runner**: Ubuntu Latest
- **Python**: 3.12
- **Node.js**: 22
- **Package Managers**: uv (Python), pnpm (Node.js, pinned to 10.26.2)

## Docker Development

The project provides Makefile commands for Docker-based development:

```bash
make docker-init    # Build images, install dependencies
make docker-start   # Start services with hot-reload
make docker-stop    # Stop containers
make docker-logs    # View logs
make up            # Production build and start
make down          # Stop and remove containers
```

## Security Notes

- All workflows use `permissions: contents: read` for minimal privilege
- Secrets are not exposed in logs
- Pull request builds cancel in-progress runs to conserve resources
- Draft PRs skip the unit test workflow to avoid unnecessary runs
