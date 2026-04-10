# Tech Stack Analysis: deer-flow

## Project Overview

DeerFlow is a LangGraph-based AI agent system with a full-stack architecture comprising frontend (Next.js), backend (Python/LangGraph), and supporting infrastructure (nginx, Docker).

## Languages and Versions

| Layer | Language | Version |
|-------|----------|---------|
| Frontend | TypeScript | 5.8.2 |
| Frontend Runtime | Node.js | 22+ |
| Backend | Python | 3.12+ |
| Build Tooling | Bash/Makefile | - |

## Frontend Stack

### Core Framework
- **Next.js 16.1.7** with App Router and Turbopack
- **React 19.0.0**
- **pnpm 10.26.2** (package manager)

### UI and Styling
- **Tailwind CSS 4.0.15** with `@tailwindcss/postcss`
- **Radix UI** primitives (dialog, dropdown, select, tooltip, etc.)
- **Class Variance Authority (CVA) 0.7.1** for component variants
- **clsx + tailwind-merge** for conditional classnames

### State and Data
- **TanStack Query 5.90.17** for server state
- **Zod 3.24.2** for runtime validation
- **`@t3-oss/env-nextjs 0.12.0`** for env validation with Zod

### AI/Streaming
- **Vercel AI SDK 6.0.33** (`ai` package) for streaming
- **LangChain Core 1.1.15** (`@langchain/core`)
- **LangGraph SDK 1.5.3** (`@langchain/langgraph-sdk`)

### Editor and Code Display
- **CodeMirror 6** (`codemirror`, `@uiw/react-codemirror`)
- **Shiki 3.15.0** for syntax highlighting
- **KaTeX** for math rendering

### Animation and Effects
- **Motion 12.26.2** (formerly Framer Motion)
- **GSAP 3.13.0** for advanced animations
- **canvas-confetti 1.9.4**

### React Component Libraries
- **embla-carousel-react 8.6.0** for carousels
- **react-resizable-panels 4.4.1** for panel layouts
- **`@xyflow/react` 12.10.0** for node-based graphs

### Internationalization
- **next-themes 0.4.6** for theme management
- `i18n/` implementation in `src/core/i18n/` (en-US, zh-CN)

### Other Notable Dependencies
- **better-auth 1.3** for authentication
- **sonner 2.0.7** for toast notifications
- **date-fns 4.1.0** for date formatting
- **nanoid 5.1.6** for ID generation
- **uuid 13.0.0** for UUID generation
- **unist-util-visit 5.0.0** for AST traversal
- **rehype-katex, rehype-raw, remark-gfm, remark-math** for Markdown processing

### Dev Dependencies
- **ESLint 9.23.0** with `typescript-eslint 8.27.0`
- **Prettier 3.5.3** with `prettier-plugin-tailwindcss 0.6.11`
- **PostCSS 8.5.3**

## Backend Stack

### Python Package Manager
- **uv 0.7.20+** (Astral's fast Python package manager)
- Workspace-based monorepo with `packages/harness/` as internal package

### Web Framework
- **FastAPI 0.115.0+**
- **Uvicorn 0.34.0+** with standard extras
- **SSE-Starlette 2.1.0** for Server-Sent Events

### LangGraph and Agent Framework
- **LangGraph 1.0.6-1.0.9** (with LangGraph API 0.7.x-0.8.x)
- **LangGraph SDK 0.1.51+**
- **LangGraph CLI 0.4.14+**
- **LangGraph Runtime In-Memory 0.22.1+**

### LangChain Integrations
- **LangChain 1.2.3+**
- Provider-specific packages:
  - `langchain-openai 1.1.7+`
  - `langchain-anthropic 1.3.4+`
  - `langchain-deepseek 1.0.1+`
  - `langchain-google-genai 4.2.1+`
- **langchain-mcp-adapters 0.1.0+** for MCP integration
- **langgraph-checkpoint-sqlite 3.0.3+** for persistence

### Sandbox and Execution
- **agent-sandbox 0.0.19+**
- **kubernetes 30.0.0+** (for K8s sandbox mode)
- **markdownify 1.2.2+**
- **markitdown[all,xlsx] 0.0.1a2+** for document conversion

### External Integrations
- **python-telegram-bot 21.0+** (Telegram channel)
- **slack-sdk 3.33.0+** (Slack channel)
- **lark-oapi 1.4.0+** (Feishu/Lark integration)
- **tavily-python 0.7.17+** (web search)
- **firecrawl-py 1.15.0+** (web scraping)
- **agent-client-protocol 0.4.0+** (ACP agents)

### Data Processing
- **pydantic 2.12.5+** for data validation
- **pyyaml 6.0.3+** for config
- **httpx 0.28.0+** for HTTP
- **tiktoken 0.8.0+** for tokenization
- **readabilipy 0.3.0+** for readability extraction
- **ddgs 9.10.0+** for DuckDuckGo
- **duckdb 1.4.4+** for embedded analytics

### Testing and Linting
- **pytest 8.0.0+**
- **ruff 0.14.11+**

## Monorepo Structure

### Workspace Configuration
```
deer-flow/
├── frontend/                 # Next.js application (pnpm workspace)
├── backend/                  # Python application (uv workspace)
│   ├── pyproject.toml        # Root backend package
│   ├── packages/
│   │   └── harness/          # deerflow-harness internal package
│   │       └── pyproject.toml
│   └── app/                  # FastAPI application (gateway + channels)
```

### Monorepo Tooling
- **uv workspace** for Python dependency management
- **pnpm workspace** for frontend (empty `packages: []` in `pnpm-workspace.yaml`)
- **No Turborepo/Nx/Lerna** - uses native package manager workspaces

## Build System

### Root Makefile (`Makefile`)
- `make check` - Verify system requirements
- `make install` - Install all dependencies (frontend + backend)
- `make dev` - Start all services in development mode
- `make start` - Start all services in production mode
- `make dev-daemon` - Background development mode
- `make stop` / `make clean` - Service cleanup
- `make docker-init/start/stop/logs` - Docker development commands
- `make up/down` - Production Docker commands

### Backend Makefile (`backend/Makefile`)
- `make install` - `uv sync`
- `make dev` - `langgraph dev` (LangGraph server on port 2024)
- `make gateway` - `uvicorn` (Gateway API on port 8001)
- `make test` - `pytest` with `PYTHONPATH=.`
- `make lint` - `ruff check`
- `make format` - `ruff check --fix && ruff format`

### Frontend Makefile (`frontend/Makefile`)
- `pnpm dev` - Development with Turbopack
- `pnpm build` - Production build
- `pnpm check` - ESLint + TypeScript check
- `pnpm lint` / `pnpm lint:fix` - Linting
- `pnpm typecheck` - TypeScript validation
- `pnpm start` - Production server

### Build Scripts
- `scripts/configure.py` - Config generation
- `scripts/serve.sh` - Service orchestration
- `scripts/start-daemon.sh` - Background service startup
- `scripts/docker.sh` - Docker management
- `scripts/deploy.sh` - Production deployment
- `scripts/cleanup-containers.sh` - Container cleanup

## Containerization

### Dockerfiles
| File | Purpose | Base Image |
|------|---------|------------|
| `frontend/Dockerfile` | Multi-stage build (dev/prod) | `node:22-alpine` |
| `backend/Dockerfile` | Backend with Python + Node | `python:3.12-slim` |
| `docker/provisioner/Dockerfile` | Provisioner service | (varies) |

### Frontend Dockerfile Stages
1. **base** - Node 22 Alpine, pnpm 10.26.2
2. **dev** - Dependencies installed, dev server
3. **builder** - Full build with `SKIP_ENV_VALIDATION=1`
4. **prod** - Minimal runtime with pre-built output

### Backend Dockerfile Features
- Python 3.12 slim with Node.js 22 (for MCP npx commands)
- Docker CLI mounted for DooD (Docker-out-of-Docker)
- `uv` package manager with caching
- Exposes ports 8001 (Gateway) and 2024 (LangGraph)

## CI/CD

### GitHub Actions Workflows

#### `backend-unit-tests.yml`
- Triggers: push to main, PR events
- Python 3.12, uv package manager
- Runs `make test` in backend directory
- 15-minute timeout
- Concurrency control (cancel-in-progress)

#### `lint-check.yml`
- **Lint job**: Python 3.12, `uv sync --group dev`, `make lint`
- **Frontend lint job**: Node 22, pnpm 10.26.2, `pnpm lint`, `pnpm typecheck`, `pnpm build`
- Builds frontend with `BETTER_AUTH_SECRET=local-dev-secret`

### Local Development Ports
| Service | Port | Purpose |
|---------|------|---------|
| Frontend | 3000 | Next.js dev server |
| Gateway API | 8001 | FastAPI REST API |
| LangGraph | 2024 | Agent runtime |
| Nginx | 2026 | Reverse proxy (entry point) |
| Provisioner | 8002 | Optional K8s mode |

## Configuration

### Main Configuration
- **config.yaml** - Application settings (models, tools, sandbox, channels, memory)
- **extensions_config.json** - MCP servers and skills
- **.env.example** - Environment variable template

### Config Versioning
- `config_version` field in `config.example.yaml`
- Auto-merge missing fields via `make config-upgrade`
- Config caching with mtime-based reload

### Environment Variable Resolution
- Config values starting with `$` resolve from environment
- Supports `DEER_FLOW_CONFIG_PATH` and `DEER_FLOW_EXTENSIONS_CONFIG_PATH`

## Key Architectural Patterns

### Harness/App Split
- **Harness** (`packages/harness/deerflow/`) - Publishable agent framework
- **App** (`app/`) - Unpublished FastAPI application
- Strict dependency direction: App imports Harness, never vice versa
- Enforced by `tests/test_harness_boundary.py`

### Thread Isolation
- Per-thread directories: `backend/.deer-flow/threads/{thread_id}/user-data/`
- Virtual path system: `/mnt/user-data/{workspace,uploads,outputs}`
- Sandbox acquisition via middleware chain

### Middleware Chain (12 middleware components)
1. ThreadDataMiddleware
2. UploadsMiddleware
3. SandboxMiddleware
4. DanglingToolCallMiddleware
5. GuardrailMiddleware
6. SummarizationMiddleware
7. TodoListMiddleware
8. TitleMiddleware
9. MemoryMiddleware
10. ViewImageMiddleware
11. SubagentLimitMiddleware
12. ClarificationMiddleware

### LangGraph Integration
- `langgraph.json` defines agent graph
- Graph entry: `deerflow.agents:make_lead_agent`
- Custom checkpointer via `async_provider.py`
