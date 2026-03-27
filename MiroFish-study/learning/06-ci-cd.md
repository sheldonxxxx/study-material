# MiroFish CI/CD

## Pipeline Overview

MiroFish uses **GitHub Actions** for Docker image builds with a single workflow. No test automation, linting, or multi-environment promotion visible in the repository.

## GitHub Actions Workflow

**File:** `.github/workflows/docker-image.yml`

**Triggers:**
- Push of any git tag
- Manual `workflow_dispatch`

**Permissions:**
- Contents: read
- Packages: write

**Jobs:**
- `build-and-push` on `ubuntu-latest`

**Steps:**
1. Checkout code
2. Set up QEMU for multi-platform support
3. Set up Docker Buildx
4. Log in to GHCR (`ghcr.io/{owner}/mirofish`)
5. Extract metadata (tags: git ref event, sha, raw=latest)
6. Build and push Docker image

**Tags Applied:**
- `type=ref,event=tag` -- e.g., `v0.1.2`
- `type=sha` -- e.g., `sha-abc1234`
- `type=raw,value=latest`

## Docker

**Dockerfile:** Multi-stage single image containing both frontend and backend.

- Base: `python:3.11`
- Installs Node.js 18+ via apt
- Copies uv from `ghcr.io/astral-sh/uv:0.9.26`
- Installs npm + Python dependencies via `uv sync --frozen`
- Exposes ports 3000 and 5001
- Entrypoint: `npm run dev`

**docker-compose.yml:** Single-service deployment.

- Image: `ghcr.io/666ghj/mirofish:latest`
- Ports: `3000:3000`, `5001:5001`
- Env file: `.env`
- Volume: `./backend/uploads:/app/backend/uploads`
- Restart policy: `unless-stopped`
- Includes commented acceleration mirror: `ghcr.nju.edu.cn/666ghj/mirofish:latest`

## Deployment Methods

| Method | Command |
|--------|---------|
| Docker Compose | `docker compose up -d` |
| Source (dev) | `npm run dev` |

## Observables

- **No test workflow** -- No pytest, no frontend tests in CI
- **No lint workflow** -- No ESLint, no Ruff, no pre-commit hooks
- **No multi-environment promotion** -- Single `latest` tag, no staging/prod gates
- **Single workflow** -- All builds go through the same docker-image.yml pipeline
