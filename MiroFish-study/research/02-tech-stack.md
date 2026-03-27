# MiroFish Tech Stack Analysis

## Overview

**MiroFish** is a "swarm intelligence engine for predicting everything" - an AI-powered simulation platform that models social media interactions and discussions through agent-based modeling (inspired by OASIS framework).

## Architecture Pattern

**Monolithic app with frontend/backend separation** - No monorepo tooling (Nx, Turborepo, Lerna). Simple npm workspace structure with two distinct services:

```
MiroFish/
├── frontend/          # Vue 3 SPA
├── backend/           # Flask API
├── Dockerfile         # Multi-stage single image
├── docker-compose.yml # Single service deployment
└── package.json       # Root orchestrator (concurrently)
```

---

## Frontend Stack

| Component | Technology | Version |
|-----------|------------|---------|
| Framework | Vue.js | 3.5.24 |
| Build Tool | Vite | 7.2.4 |
| Language | TypeScript | ESM module |
| HTTP Client | Axios | 1.13.2 |
| Visualization | D3.js | 7.9.0 |
| Router | Vue Router | 4.6.3 |
| Plugin | @vitejs/plugin-vue | 6.0.1 |

**Dev Server Configuration:**
- Port: 3000
- Proxy: `/api` -> `http://localhost:5001` (Flask backend)

---

## Backend Stack

| Component | Technology | Version |
|-----------|------------|---------|
| Framework | Flask | 3.0.0+ |
| Language | Python | 3.11+ |
| Package Manager | uv | (via pyproject.toml) |
| CORS | flask-cors | 6.0.0+ |
| Data Validation | Pydantic | 2.0.0+ |
| File Processing | PyMuPDF | 1.24.0+ |
| LLM SDK | OpenAI SDK | 1.0.0+ |
| Memory Graph | Zep Cloud | 3.13.0 |
| Social Sim | CAMEL-OASIS | 0.2.5 |
| Multi-Agent | CAMEL-AI | 0.2.78 |

**Dev tooling:**
- pytest, pytest-asyncio for testing
- hatchling for packaging

**Runtime Configuration:**
- Host: 0.0.0.0
- Port: 5001
- Debug mode available

---

## Build & Dev Orchestration

### Root package.json Scripts

| Script | Command |
|--------|---------|
| `npm run dev` | `concurrently` - runs both backend + frontend |
| `npm run backend` | `cd backend && uv run python run.py` |
| `npm run frontend` | `cd frontend && npm run dev` |
| `npm run build` | `cd frontend && vite build` |
| `npm run setup` | `npm install + cd frontend && npm install` |
| `npm run setup:backend` | `cd backend && uv sync` |

### Package Managers

| Layer | Manager | Lock File |
|-------|---------|-----------|
| Root | npm | package-lock.json |
| Frontend | npm | frontend/package-lock.json |
| Backend | uv | backend/uv.lock |

---

## Infrastructure

### Docker

**Base Image:** `python:3.11` with Node.js 18+ installed via apt

**Multi-runtime container** that installs:
1. Node.js/npm via apt
2. Python dependencies via uv
3. All project dependencies

**Exposed Ports:** 3000 (frontend), 5001 (backend)

**Entrypoint:** `npm run dev` (uses concurrently to run both services)

### CI/CD

**GitHub Actions** (`docker-image.yml`):
- Trigger: git tags + manual workflow_dispatch
- Multi-platform build with QEMU
- Push to GHCR: `ghcr.io/{owner}/mirofish`
- Tags: `latest`, `sha-*`, git ref-based

### Environment Variables

```
LLM_API_KEY, LLM_BASE_URL, LLM_MODEL_NAME     # OpenAI-compatible LLM
ZEP_API_KEY                                   # Memory graph service
LLM_BOOST_*                                   # Optional acceleration LLM
FLASK_HOST, FLASK_PORT                        # Backend runtime config
```

---

## Key Observations

1. **No database** - Backend appears stateless, processing happens in-memory or via external services (Zep Cloud)

2. **Two LLM integrations** - Primary LLM (Alibaba Bailian/qwen-plus recommended) + optional boost LLM for acceleration

3. **Agent-based simulation** - Uses CAMEL-OASIS framework for social media simulation, CAMEL-AI for multi-agent coordination

4. **Simple deployment** - Single Docker image contains both frontend and backend; no Kubernetes or complex orchestration

5. **Windows compatibility** - Backend code includes Windows-specific UTF-8 encoding fixes

6. **uv for Python** - Modern fast Python package manager (astral-sh) instead of pip; uses pyproject.toml

7. **No TypeScript in backend** - Python backend uses pydantic for typed config, no FastAPI/typer

---

## File Structure

```
MiroFish/
├── .github/workflows/docker-image.yml
├── .env.example
├── Dockerfile
├── docker-compose.yml
├── package.json                    # Root - npm workspaces + concurrently
├── package-lock.json
│
├── frontend/
│   ├── package.json                # npm
│   ├── package-lock.json
│   ├── vite.config.js              # Vite with /api proxy
│   ├── index.html
│   ├── public/
│   └── src/
│
├── backend/
│   ├── pyproject.toml              # uv + hatchling
│   ├── requirements.txt            # pip fallback
│   ├── uv.lock
│   ├── run.py                      # Flask entrypoint
│   ├── app/
│   │   ├── __init__.py
│   │   ├── config.py
│   │   ├── api/
│   │   ├── models/
│   │   ├── services/
│   │   └── utils/
│   └── scripts/
│
└── static/                         # (empty or static assets)
```
