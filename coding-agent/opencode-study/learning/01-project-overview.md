# OpenCode Project Overview

## What is OpenCode?

OpenCode is an **AI-powered coding assistant** that runs locally on the developer's machine. It provides a CLI interface and multiple frontend applications for interacting with AI models to assist with software development tasks.

**Repository:** `/Users/sheldon/Documents/claw/reference/opencode`
**Package Manager:** Bun (v1.3.11)
**Architecture:** Monorepo with Turborepo
**Main Language:** TypeScript

---

## Core Value Proposition

OpenCode serves as an AI agent that can:

- Read, edit, and create files in a codebase
- Execute shell commands and bash scripts
- Search code using glob patterns and grep
- Integrate with Language Server Protocol (LSP) for deep code understanding
- Interface with external services via MCP (Model Context Protocol)
- Manage git operations and pull requests
- Provide web search and web fetching capabilities

The agent can operate autonomously or via interactive chat, executing tasks based on natural language instructions.

---

## Project Scope

### Primary Deliverable

**`packages/opencode`** - The main CLI application built with Yargs. Entry point: `packages/opencode/src/index.ts`

### Multi-Platform Deployment

| Platform | Package | Technology |
|----------|---------|------------|
| CLI | `opencode` | Bun/Node.js (Yargs) |
| Desktop | `desktop` | Tauri + Solid.js + Rust |
| Web App | `app` | SolidStart |
| Marketing | `web` | Astro |
| Documentation | `docs` | Custom |

### Integration Ecosystem

- **GitHub:** GitHub App integration for repository operations
- **Slack:** Slack integration for notifications
- **MCP:** Model Context Protocol server implementation
- **Plugin:** Extensible plugin system

---

## Monorepo Structure

```
opencode/
├── packages/              # 19 packages
│   ├── opencode/          # Main CLI
│   ├── app/               # SolidStart web app
│   ├── console/           # SST backend services
│   ├── desktop/           # Tauri desktop app
│   ├── web/               # Astro marketing site
│   ├── ui/                # Shared UI components
│   ├── sdk/               # JavaScript SDK
│   └── ...                # 12 more packages
├── github/                # GitHub App infrastructure
├── infra/                 # Infrastructure as code
├── script/                # Build/release scripts
├── sdks/                  # SDK implementations
└── specs/                 # Specifications
```

### Key Package Directories

| Package | Purpose |
|---------|---------|
| `opencode/src/agent/` | Agent core (LLM interaction, message processing, prompts) |
| `opencode/src/tool/` | 47+ tools (bash, read, write, edit, glob, grep, LSP, webfetch, etc.) |
| `opencode/src/server/` | HTTP server with routes |
| `opencode/src/provider/` | AI provider abstraction |
| `opencode/src/mcp/` | MCP server implementation |

---

## Technology Stack

### Backend/CLI

- **Runtime:** Bun, Node.js
- **CLI Framework:** Yargs
- **HTTP Server:** Hono
- **Database:** SQLite with Drizzle ORM
- **Auth:** OpenAuth.js (OAuth), API keys
- **AI SDK:** Vercel AI SDK
- **Structured Concurrency:** Effect framework

### Frontend

- **UI Framework:** Solid.js, SolidStart
- **Desktop:** Tauri (Rust backend), Electron (legacy)
- **Web:** Astro
- **Styling:** Tailwind CSS

### Development

- **Build:** Turborepo
- **Package Manager:** Bun
- **Type System:** TypeScript with Zod schemas
- **Testing:** bun:test, Playwright

---

## Entry Points

### CLI Commands

```bash
opencode run           # Run agent in current directory
opencode chat          # Chat interface
opencode agent         # Agent management
opencode serve         # Start server mode
opencode providers     # AI provider management
opencode models        # Model management
opencode mcp           # MCP tools
opencode github        # GitHub integration
opencode pr            # Pull request tools
opencode db            # Database tools
```

### Development Modes

```bash
bun dev                # CLI development
bun dev:desktop        # Tauri desktop dev
bun dev:app            # SolidStart app dev
bun dev:console        # Console backend dev
```

---

## Agent Architecture

The agent system in `packages/opencode/src/agent/` comprises:

| Module | Responsibility |
|--------|----------------|
| `index.ts` | Main orchestrator |
| `llm.ts` | LLM interaction |
| `message.ts` | Message handling |
| `processor.ts` | Message processing pipeline |
| `prompt.ts` | Prompt generation (70KB+) |
| `compaction.ts` | Context window management |
| `retry.ts` | Failed operation retry |
| `revert.ts` | Operation revert handling |
| `summary.ts` | Conversation summarization |

---

## Configuration System

OpenCode uses a layered configuration system (low to high precedence):

1. Remote `.well-known/opencode` (organization defaults)
2. Global config `~/.config/opencode/opencode.json`
3. Custom config via `OPENCODE_CONFIG`
4. Project config `opencode.json`
5. `.opencode` directories
6. Inline config via `OPENCODE_CONFIG_CONTENT`
7. Enterprise managed config directories

---

## Summary

OpenCode is a mature, production-ready monorepo that delivers AI-assisted coding through:

- **A powerful CLI** for terminal-based workflows
- **Multiple deployment targets** (desktop, web, mobile-ready)
- **Rich tool ecosystem** with 47+ built-in tools
- **Deep integrations** with GitHub, Slack, and MCP
- **Enterprise-ready** architecture with plugin system

The project demonstrates professional software engineering with comprehensive testing, typed schemas, structured concurrency, and pragmatic security patterns.
