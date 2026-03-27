# Dependencies

## Overview

DeerFlow uses a monorepo structure with two package managers: **pnpm** (frontend) and **uv** (backend Python). The project maintains strict dependency boundaries between the harness framework and application layers.

## Frontend Dependencies

**Package Manager**: pnpm 10.26.2 (pinned)

### Core Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `next` | 16.1.7 | React framework |
| `react` / `react-dom` | 19.0.0 | UI library |
| `typescript` | 5.8.2 | Type safety |
| `@tanstack/react-query` | 5.90.17 | Server state management |
| `ai` (Vercel) | 6.0.33 | AI streaming SDK |
| `zod` | 3.24.2 | Schema validation |

### UI Components

| Package | Purpose |
|---------|---------|
| `@radix-ui/react-*` | Unstyled accessible primitives (dialog, dropdown, tabs, tooltip, etc.) |
| `class-variance-authority` | Component variant management |
| `clsx` / `tailwind-merge` | Conditional className utility |
| `tailwindcss` | 4.0.15 - Styling |
| `lucide-react` | Icon library |
| `sonner` | Toast notifications |
| `embla-carousel-react` | Carousel |
| `react-resizable-panels` | Resizable panels |
| `@xyflow/react` | Diagram rendering |
| `motion` / `gsap` | Animations |

### Editor & Content

| Package | Purpose |
|---------|---------|
| `codemirror` | 6.0.2 - Code editor |
| `@codemirror/lang-*` | Language support (JS, Python, JSON, Markdown, CSS, HTML) |
| `@uiw/react-codemirror` | React CodeMirror wrapper |
| `shiki` | 3.15.0 - Syntax highlighting |
| `rehype-katex` / `remark-math` | Math rendering |
| `remark-gfm` / `rehype-raw` | Markdown processing |
| `hast` / `@types/hast` | HAST utilities |

### Auth & API

| Package | Purpose |
|---------|---------|
| `better-auth` | 1.3 - Authentication |
| `@t3-oss/env-nextjs` | Environment validation |
| `@langchain/langgraph-sdk` | LangGraph client |

### Utilities

| Package | Purpose |
|---------|---------|
| `nanoid` | ID generation |
| `date-fns` | Date formatting |
| `uuid` | UUID generation |
| `canvas-confetti` | Confetti effects |
| `katex` | Math typesetting |
| `dotenv` | Environment loading |

### Dev Dependencies

- `eslint` / `eslint-config-next`
- `prettier` / `prettier-plugin-tailwindcss`
- `@types/node` / `@types/react` / `@types/react-dom`
- `tailwindcss` / `@tailwindcss/postcss`
- `tw-animate-css`

## Backend Dependencies

**Package Manager**: uv with Python 3.12+

### Workspace Structure

```
backend/
├── pyproject.toml              # Main app (deer-flow)
└── packages/
    └── harness/
        └── pyproject.toml      # Framework (deerflow-harness)
```

### Harness Package (`deerflow-harness`)

The core agent framework with all agent capabilities.

| Package | Version | Purpose |
|---------|---------|---------|
| `langgraph` | 1.0.6-1.0.9 | Agent orchestration |
| `langgraph-api` | 0.7.0-0.8.0 | API compatibility |
| `langgraph-cli` | 0.4.14+ | Server CLI |
| `langgraph-runtime-inmem` | 0.22.1+ | In-memory execution |
| `langgraph-checkpoint-sqlite` | 3.0.3+ | State persistence |
| `langgraph-sdk` | 0.1.51+ | Client SDK |
| `langchain` | 1.2.3+ | LLM framework |
| `langchain-openai` | 1.1.7+ | OpenAI integration |
| `langchain-anthropic` | 1.3.4+ | Anthropic integration |
| `langchain-deepseek` | 1.0.1+ | DeepSeek integration |
| `langchain-google-genai` | 4.2.1+ | Google GenAI integration |
| `langchain-mcp-adapters` | 0.1.0+ | MCP server integration |
| `agent-client-protocol` | 0.4.0+ | Agent communication |
| `agent-sandbox` | 0.0.19+ | Sandbox framework |
| `pydantic` | 2.12.5+ | Data validation |
| `pyyaml` | 6.0.3+ | Config parsing |
| `httpx` | 0.28.0+ | HTTP client |
| `kubernetes` | 30.0.0+ | K8s client |
| `tavily-python` | 0.7.17+ | Web search |
| `firecrawl-py` | 1.15.0+ | Web scraping |
| `tiktoken` | 0.8.0+ | Token counting |
| `ddgs` | 9.10.0+ | DuckDuckGo search |
| `duckdb` | 1.4.4+ | SQL database |
| `markdownify` | 1.2.2+ | Markdown conversion |
| `markitdown[all,xlsx]` | 0.0.1a2+ | Document conversion |
| `readabilipy` | 0.3.0+ | Readability extraction |
| `dotenv` | 0.9.9+ | Env loading |

### Application Package (`deer-flow`)

REST API and integrations layer.

| Package | Purpose |
|---------|---------|
| `fastapi` | 0.115.0+ Web framework |
| `uvicorn[standard]` | 0.34.0+ ASGI server |
| `httpx` | 0.28.0+ HTTP client |
| `sse-starlette` | 2.1.0+ Server-sent events |
| `python-multipart` | 0.0.20+ Form data |
| `langgraph-sdk` | 0.1.51+ LangGraph client |
| `lark-oapi` | 1.4.0+ Feishu API |
| `slack-sdk` | 3.33.0+ Slack API |
| `python-telegram-bot` | 21.0+ Telegram bot |
| `markdown-to-mrkdwn` | 1.2.2+ Markdown conversion |

### Dev Dependencies

- `pytest` - Testing
- `ruff` - Linting/formatting

## Dependency Groups

### Backend (`uv sync --group dev`)
- `pytest>=8.0.0`
- `ruff>=0.14.11`

## Key Integration Points

### LLM Providers (via LangChain)

- OpenAI (including Responses API)
- Anthropic (Claude)
- Google GenAI (Gemini)
- DeepSeek
- OpenRouter
- Azure OpenAI (compatible via base_url)

### Community Tools

- Tavily (web search)
- Firecrawl (web scraping)
- Jina AI (readability)
- DuckDuckGo (image search)

### Messaging Platforms

- Telegram (python-telegram-bot)
- Slack (slack-sdk with Socket Mode)
- Feishu/Lark (lark-oapi with WebSocket)

## Security Considerations

- **Sandbox Isolation**: Code execution isolated in Docker/K8s containers
- **Config Security**: API keys recommended via environment variables
- **Local Deployment**: Designed for localhost by default
- **Path Translation**: Virtual paths prevent directory traversal
