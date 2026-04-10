# deer-flow Directory Structure and Entry Points

## Root Level Overview

```
deer-flow/
├── backend/              # Python FastAPI application
├── frontend/             # Next.js React application
├── docker/               # Docker configuration files
├── scripts/              # Shell/Python utility scripts
├── docs/                 # Documentation
├── skills/               # Skills directory
├── .github/              # GitHub workflows
├── Makefile              # Root make targets
├── config.example.yaml   # Example configuration
└── README*.md            # Multi-language documentation
```

## Backend Structure (Python/FastAPI)

```
backend/
├── app/
│   ├── __init__.py
│   ├── app.py                    # FastAPI application entry point
│   ├── config.py                 # Configuration loader
│   ├── path_utils.py
│   ├── channels/                 # Messaging channel integrations
│   │   ├── base.py
│   │   ├── manager.py            # Channel manager
│   │   ├── message_bus.py
│   │   ├── service.py
│   │   ├── store.py
│   │   ├── feishu.py             # Feishu (Lark) integration
│   │   ├── slack.py             # Slack integration
│   │   └── telegram.py           # Telegram integration
│   └── gateway/
│       ├── __init__.py
│       ├── layout.tsx            # Next.js layout (API proxy?)
│       ├── page.tsx              # Next.js page
│       ├── mock/
│       ├── workspace/
│       └── routers/              # API route handlers
│           ├── agents.py
│           ├── artifacts.py
│           ├── channels.py
│           ├── mcp.py            # Model Context Protocol
│           ├── memory.py
│           ├── models.py
│           ├── skills.py
│           ├── suggestions.py
│           ├── threads.py
│           └── uploads.py
├── packages/
│   └── harness/                  # Testing harness
├── tests/                        # Test suite
├── pyproject.toml
├── uv.lock
└── Dockerfile
```

**Backend Entry Points:**
- `app/app.py` - FastAPI application factory/entry point
- `app/config.py` - Configuration loader

## Frontend Structure (Next.js/React)

```
frontend/
├── src/
│   ├── app/                       # Next.js App Router
│   │   ├── api/                   # API routes
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── components/                # React components
│   ├── core/                      # Core business logic (22 dirs)
│   ├── hooks/                     # Custom React hooks
│   ├── lib/                       # Utility libraries
│   ├── server/                    # Server-side code
│   ├── styles/                    # CSS/styles
│   ├── typings/                   # TypeScript type definitions
│   └── env.js                     # Environment configuration
├── public/                        # Static assets
├── next.config.js
├── package.json
└── tsconfig.json
```

**Frontend Entry Points:**
- `src/app/page.tsx` - Main page component
- `src/app/layout.tsx` - Root layout
- `next.config.js` - Next.js configuration

## Docker Structure

```
docker/
├── docker-compose.yaml            # Production compose
├── docker-compose-dev.yaml        # Development compose
├── nginx/                         # Nginx configuration
└── provisioner/                   # Provisioning scripts
```

## Scripts

```
scripts/
├── check.sh / check.py            # Validation scripts
├── cleanup-containers.sh
├── config-upgrade.sh
├── configure.py
├── deploy.sh
├── docker.sh
├── export_claude_code_oauth.py
├── serve.sh
├── start-daemon.sh
├── tool-error-degradation-detection.sh
└── wait-for-port.sh
```

## Key Observations

1. **Monorepo Structure**: Project is a monorepo with separate `backend/` (Python) and `frontend/` (TypeScript/Next.js) directories.

2. **Backend Pattern**: Python FastAPI backend with:
   - `app/` containing the main application
   - `channels/` for multi-platform messaging integrations (Slack, Telegram, Feishu)
   - `gateway/` appears to contain Next.js pages (possibly for API proxying or embedded UI)

3. **Frontend Pattern**: Next.js with App Router, featuring:
   - `src/core/` with extensive business logic (22 subdirectories)
   - `src/components/` for UI components
   - TypeScript throughout

4. **Multi-Channel**: Backend supports multiple messaging channels (Slack, Telegram, Feishu/Lark)

5. **Testing**: Backend has `packages/harness/` for testing and `tests/` directory

6. **Configuration**: Centralized config via `config.example.yaml` at root
