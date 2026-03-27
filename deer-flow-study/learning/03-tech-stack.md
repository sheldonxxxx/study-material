# Tech Stack

## Overview

DeerFlow is a full-stack AI agent application built with modern, production-ready technologies across frontend, backend, and infrastructure layers. The project emphasizes extensibility, observability, and developer experience.

## Frontend Stack

| Layer | Technology | Version |
|-------|------------|---------|
| **Framework** | Next.js | 16.1.7 |
| **UI Library** | React | 19.0.0 |
| **Language** | TypeScript | 5.8.2 |
| **Styling** | Tailwind CSS | 4.0.15 |
| **State Management** | TanStack Query | 5.90.17 |
| **AI Streaming** | Vercel AI SDK | 6.0.33 |
| **Code Editor** | CodeMirror | 6.0.2 |
| **Diagrams** | @xyflow/react | 12.10.0 |
| **Animation** | Motion, GSAP | 12.26.2, 3.13.0 |
| **Auth** | better-auth | 1.3 |
| **Validation** | Zod | 3.24.2 |
| **Package Manager** | pnpm | 10.26.2 |

### Frontend Key Libraries

- **Radix UI** - Unstyled, accessible UI primitives (dialog, dropdown, tabs, tooltip, etc.)
- **Shiki** (3.15.0) - Syntax highlighting
- **KaTeX** - Math rendering
- **remark-gfm / rehype-katex** - Markdown with GitHub Flavored Markdown support
- **sonner** - Toast notifications
- **embla-carousel-react** - Carousel component
- **react-resizable-panels** - Resizable panel layouts
- **codemirror** - Code editing with language support for JS, Python, JSON, Markdown, CSS, HTML

## Backend Stack

| Layer | Technology | Version |
|-------|------------|---------|
| **Runtime** | Python | 3.12+ |
| **Agent Framework** | LangGraph | 1.0.6-1.0.9 |
| **LLM Integration** | LangChain | 1.2.3+ |
| **API Framework** | FastAPI | 0.115.0+ |
| **Server** | Uvicorn | 0.34.0+ |
| **HTTP Client** | httpx | 0.28.0+ |
| **Validation** | Pydantic | 2.12.5+ |
| **Config** | PyYAML | 6.0.3+ |
| **Package Manager** | uv | latest |

### Backend Key Libraries

- **langgraph-sdk** - LangGraph server client
- **langgraph-cli** - LangGraph server CLI
- **langgraph-api** - LangGraph API compatibility
- **langchain-mcp-adapters** - MCP server integration
- **kubernetes** - K8s client for sandbox provisioning
- **python-telegram-bot** - Telegram channel integration
- **slack-sdk** - Slack channel integration
- **lark-oapi** - Feishu/Lark channel integration
- **markdown-to-mrkdwn** - Markdown conversion
- **markitdown** - Document conversion (PDF, Excel, Word, PPT)
- **readabilipy** - Readability extraction
- **tiktoken** - Token counting
- **firecrawl-py** - Web scraping
- **tavily-python** - Web search

### Agent & Tooling

- **agent-client-protocol (ACP)** - Agent communication protocol
- **agent-sandbox** - Sandbox execution framework
- **langgraph-checkpoint-sqlite** - State persistence

## Infrastructure

| Component | Technology |
|-----------|------------|
| **Reverse Proxy** | nginx |
| **Containerization** | Docker |
| **Orchestration** | Docker Compose |
| **Sandbox** | Docker containers, Kubernetes (optional) |
| **Package Registry** | GitHub Packages |

### Service Architecture

```
Browser
  ↓
Nginx (port 2026) ← Unified entry point
  ├→ Frontend (port 3000) ← Next.js with hot-reload
  ├→ Gateway API (port 8001) ← FastAPI REST API
  └→ LangGraph Server (port 2024) ← Agent runtime
      └→ Sandboxed Execution Environment ← Docker/K8s
```

## Development Tools

### Frontend
- **ESLint** - Linting with typescript-eslint
- **Prettier** - Code formatting with tailwindcss plugin
- **TypeScript** - Type checking

### Backend
- **ruff** - Linting and formatting (line length: 240)
- **pytest** - Testing framework
- **uv** - Python package manager with workspace support

## Summary

DeerFlow leverages a well-established modern tech stack:

- **Frontend**: Next.js + React 19 + TypeScript for type-safe UI
- **Backend**: Python + LangGraph + LangChain for agent orchestration
- **API**: FastAPI for REST endpoints
- **Execution**: Docker/K8s sandboxes for isolated code execution
- **Integrations**: MCP for tool extensibility, ACP for agent-to-agent communication

The stack prioritizes production-readiness with strong typing, comprehensive testing, and containerized deployment options.
