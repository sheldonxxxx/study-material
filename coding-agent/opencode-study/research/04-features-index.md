# OpenCode Feature Index

## Project Overview

OpenCode is an open-source AI coding agent that runs in the terminal (TUI) with a client/server architecture. It supports multiple AI providers (Anthropic, OpenAI, Google, AWS Bedrock, local models) and can be extended via plugins, SDK, and VS Code extension.

---

## Core Features

### 1. AI Coding Agent (Core Engine)
**Priority: Core**
**Description:** The main AI-powered coding agent that understands context, edits files, runs commands, and assists with development tasks. Supports multiple specialized subagents.
**Key Files/Locations:**
- `packages/opencode/src/agent/` - Agent subsystem
- `packages/opencode/src/index.ts` - Entry point
- `packages/opencode/src/server/` - Server implementation
- `.opencode/agent/` - Agent configuration
**Artifacts:**
- `AGENTS.md` - Documents build/plan/general subagents
- Effect-based architecture (`packages/opencode/src/effect/`)

---

### 2. Multi-Provider AI Support
**Priority: Core**
**Description:** Provider-agnostic design supporting Anthropic Claude, OpenAI GPT, Google Gemini, AWS Bedrock, Cerebras, Cohere, local models, and more via unified AI SDK.
**Key Files/Locations:**
- `packages/opencode/src/provider/` - Provider abstraction layer
- `packages/opencode/package.json` - AI SDK dependencies (@ai-sdk/*)
- `packages/opencode/src/acp/` - Agent Client Protocol implementation
**Notes:** Uses `ai` SDK (Vercel) as the core AI abstraction with provider-specific adapters.

---

### 3. Built-in Agent System (build/plan/general)
**Priority: Core**
**Description:** Three specialized subagents switchable via Tab key: build (full development access), plan (read-only analysis), and general (complex multi-step searches).
**Key Files/Locations:**
- `packages/opencode/AGENTS.md` - Agent documentation
- `packages/opencode/src/agent/` - Agent implementations
- `.opencode/command/` - Agent command definitions

---

### 4. Language Server Protocol (LSP) Support
**Priority: Core**
**Description:** Out-of-the-box LSP support for code intelligence including hover, goto definition, completions, and diagnostics.
**Key Files/Locations:**
- `packages/opencode/src/lsp/` - LSP implementation
- `packages/opencode/src/ide/` - IDE integration layer
**Notes:** Built-in vscode-jsonrpc for LSP communication.

---

### 5. Terminal User Interface (TUI)
**Priority: Core**
**Description:** Terminal-based UI built by neovim users, featuring terminal.shop heritage. Rich interactive CLI experience with real-time output.
**Key Files/Locations:**
- `packages/opencode/src/pty/` - PTY handling (bun-pty)
- `packages/opencode/src/bus/` - Event bus for UI updates
- `packages/opencode/src/shell/` - Shell integration
**Notes:** Client/server architecture allows TUI frontend to run remotely while driving the backend.

---

### 6. Desktop Application (Tauri v2)
**Priority: Core**
**Description:** Native cross-platform desktop app (macOS, Windows, Linux) built with Tauri v2, available as DMG, EXE, deb, rpm, AppImage.
**Key Files/Locations:**
- `packages/desktop/` - Tauri desktop app
- `packages/desktop/src-tauri/` - Rust backend
- `packages/desktop-electron/` - Electron variant

---

### 7. Client/Server Architecture
**Priority: Core**
**Description:** Distributed architecture allowing OpenCode server to run on one machine while controlled remotely from another, enabling mobile app clients and remote development.
**Key Files/Locations:**
- `packages/opencode/src/server/` - Server implementation
- `packages/opencode/src/client/` - Client implementation
- `packages/opencode/src/connection/` - Connection management

---

## Secondary Features

### 8. VS Code Extension
**Priority: Secondary**
**Description:** VS Code extension providing OpenCode integration directly in the popular IDE.
**Key Files/Locations:**
- `sdks/vscode/` - VS Code extension source
- `sdks/vscode/src/` - Extension implementation

---

### 9. SDK for External Integrations
**Priority: Secondary**
**Description:** Public SDK (`@opencode-ai/sdk`) for building external tools and integrations that interact with OpenCode.
**Key Files/Locations:**
- `packages/sdk/` - SDK package
- `packages/sdk/js/` - JavaScript/TypeScript SDK
- `packages/sdk/openapi.json` - OpenAPI spec for REST API

---

### 10. Web Dashboard (SolidJS)
**Priority: Secondary**
**Description:** Web-based dashboard/app for OpenCode, built with SolidJS.
**Key Files/Locations:**
- `packages/app/` - SolidJS application
- `packages/app/src/` - App source code
- `packages/app/e2e/` - Playwright E2E tests

---

### 11. Documentation Website
**Priority: Secondary**
**Description:** Official documentation at opencode.ai/docs, built with Astro Starlight.
**Key Files/Locations:**
- `packages/web/` - Astro Starlight site
- `packages/web/src/content/docs/` - Documentation content
- `packages/docs/` - Additional docs

---

### 12. Enterprise Features
**Priority: Secondary**
**Description:** Enterprise tier with authentication, access control, and organizational management.
**Key Files/Locations:**
- `packages/enterprise/` - Enterprise package
- `packages/identity/` - Identity/auth assets
- `packages/opencode/src/auth/` - Auth subsystem
- `infra/enterprise.ts` - Enterprise infrastructure

---

### 13. Slack Integration
**Priority: Secondary**
**Description:** Slack app for team collaboration and notifications.
**Key Files/Locations:**
- `packages/slack/` - Slack integration
- `packages/slack/src/` - Slack app implementation

---

### 14. Plugin System
**Priority: Secondary**
**Description:** Extensible plugin architecture for adding custom functionality.
**Key Files/Locations:**
- `packages/plugin/` - Plugin system
- `packages/opencode/src/plugin/` - Plugin integration in core
- `packages/opencode/src/mcp/` - Model Context Protocol support

---

### 15. Database & Storage
**Priority: Secondary**
**Description:** Drizzle ORM-based database with Bun runtime support for persistent state.
**Key Files/Locations:**
- `packages/opencode/src/storage/` - Storage implementations
- `packages/opencode/src/sql.d.ts` - SQL type definitions
- `packages/opencode/drizzle.config.ts` - Drizzle configuration
- `packages/opencode/migration/` - Database migrations

---

### 16. Git Integration
**Priority: Secondary**
**Description:** Deep Git integration for version control operations.
**Key Files/Locations:**
- `packages/opencode/src/git/` - Git operations
- `packages/opencode/git/` - Git hooks/scripts
- `packages/opencode/src/worktree/` - Git worktree management

---

### 17. Cloud Infrastructure (SST)
**Priority: Secondary**
**Description:** AWS-based cloud infrastructure using SST (Serverless Stack).
**Key Files/Locations:**
- `infra/` - SST infrastructure definitions
- `infra/app.ts` - Main app infrastructure
- `infra/console.ts` - Console infrastructure
- `sst.config.ts` - SST configuration

---

### 18. Container Images
**Priority: Secondary**
**Description:** Docker/container images for server deployment.
**Key Files/Locations:**
- `packages/containers/` - Container definitions
- `packages/containers/base/` - Base image
- `packages/containers/bun-node/` - Bun Node image
- `packages/containers/rust/` - Rust runtime image
- `packages/containers/tauri-linux/` - Tauri Linux image

---

## Summary Table

| Feature | Priority | Key Location |
|---------|----------|--------------|
| AI Coding Agent | Core | `packages/opencode/src/agent/` |
| Multi-Provider Support | Core | `packages/opencode/src/provider/` |
| Agent System (build/plan/general) | Core | `packages/opencode/AGENTS.md` |
| LSP Integration | Core | `packages/opencode/src/lsp/` |
| Terminal UI (TUI) | Core | `packages/opencode/src/pty/` |
| Desktop App (Tauri) | Core | `packages/desktop/` |
| Client/Server Architecture | Core | `packages/opencode/src/server/` |
| VS Code Extension | Secondary | `sdks/vscode/` |
| SDK | Secondary | `packages/sdk/` |
| Web Dashboard | Secondary | `packages/app/` |
| Documentation Site | Secondary | `packages/web/` |
| Enterprise Features | Secondary | `packages/enterprise/` |
| Slack Integration | Secondary | `packages/slack/` |
| Plugin System | Secondary | `packages/plugin/` |
| Database/Storage | Secondary | `packages/opencode/src/storage/` |
| Git Integration | Secondary | `packages/opencode/src/git/` |
| Cloud Infrastructure | Secondary | `infra/` |
| Container Images | Secondary | `packages/containers/` |

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                 │
├─────────────────┬──────────────────┬────────────────────────────┤
│  Terminal (TUI)│  Desktop (Tauri) │  VS Code Extension         │
│  packages/opencode│  packages/desktop │  sdks/vscode            │
└────────┬────────┴────────┬─────────┴──────────────┬─────────────┘
         │                 │                        │
         └─────────────────┴────────────────────────┘
                           │ RPC/WebSocket
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SERVER (packages/opencode)                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │   Agent     │  │   LSP       │  │   Provider (AI SDK)     │  │
│  │  Subsystem  │  │  Server     │  │  Anthropic/OpenAI/etc   │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │   Git       │  │  Database   │  │   Plugin/MCP           │  │
│  │  Worktree   │  │  (Drizzle)  │  │   Extensions           │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                   CLOUD (infra/, packages/console)              │
│  AWS Infrastructure (SST)  │  Enterprise Auth  │  Slack       │
└─────────────────────────────────────────────────────────────────┘
```
