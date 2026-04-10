# OpenClaw Architecture

**Project:** OpenClaw - Personal AI Assistant
**Source:** `/Users/sheldon/Documents/claw/reference/openclaw`
**Date:** 2026-03-26
**Version:** 2026.3.24

---

## 1. Architectural Overview

OpenClaw is a **multi-channel AI gateway** that connects messaging platforms to an AI agent runtime. The system runs locally on user devices as a self-hosted control plane, supporting 20+ messaging channels simultaneously.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Messaging Channels                                     │
│  WhatsApp | Telegram | Slack | Discord | Signal | iMessage | Matrix | ...  │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      OpenClaw Gateway                                        │
│              (Hono + Express + WebSocket on ws://127.0.0.1:18789)            │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐ │
│  │  Sessions   │ │  Channels   │ │   Tools     │ │   Cron / Webhooks /MCP  │ │
│  │             │ │             │ │  Browser    │ │                         │ │
│  │             │ │             │ │  Canvas     │ │                         │ │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────────┬──────────────┘ │
│         └───────────────┴───────────────┘                     │                │
│                             │                                 │                │
│                    ┌────────▼────────┐                        │                │
│                    │   Pi Agent     │                        │                │
│                    │   (Runtime)    │                        │                │
│                    └────────────────┘                        │                │
└─────────────────────────────────┬────────────────────────────┴────────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        │                         │                         │
        ▼                         ▼                         ▼
┌───────────────┐         ┌───────────────┐         ┌───────────────┐
│   macOS App   │         │   iOS Node    │         │ Android Node  │
│  Menu bar UI  │         │   Canvas      │         │  Chat/Voice   │
│  Voice Wake   │         │   Voice Wake  │         │  Camera/Screen│
│  WebChat      │         │   Camera     │         │  Device Cmds  │
└───────────────┘         └───────────────┘         └───────────────┘
```

---

## 2. Core Subsystems

### 2.1 Gateway (`src/gateway/`)

The Gateway is the central server that orchestrates all components. It is built on **Hono** (HTTP framework) with **Express** compatibility and **WebSocket** support.

**Key Files:**
- `server.impl.ts` - Main server implementation (1400+ lines)
- `server-http.ts` - HTTP request handling (routes, hooks, auth)
- `server-ws-runtime.ts` - WebSocket runtime setup
- `client.ts` - Gateway client for external connections
- `server-chat.ts` - Chat session handling
- `server-methods.ts` - RPC method handlers

**Gateway Responsibilities:**
- Session management with presence and typing indicators
- Channel coordination and message routing
- Tool execution coordination (browser, canvas, etc.)
- Cron jobs and webhook triggers
- Authentication and access control
- Configuration management

**Port:** `18789` (localhost by default)

### 2.2 Pi Agent Runtime (`src/agents/`)

The Pi Agent is the core AI processing engine. It processes messages, executes tools, and coordinates responses.

**Key Components:**
- `acp-spawn.ts` - Agent spawning logic
- `agent-command.ts` - Command handling (~43KB)
- `agent-scope.ts` - Session scoping
- `pi-embedded-runner/` - Embedded agent execution

**Agent Features:**
- RPC-based agent runtime
- Tool streaming and block streaming
- Multi-agent routing (route channels/accounts to isolated agents)
- Session model: main for direct chats, group isolation
- Context compaction and session pruning
- Reply-back and queue modes

### 2.3 ACP Protocol (`src/acp/`)

The **Agent Client Protocol (ACP)** is the wire protocol for IDE-to-gateway communication.

**Key Files:**
- `session.ts` - Session management
- `client.ts` - ACP client implementation
- `control-plane/` - Control plane implementation
  - `manager.core.ts` - Core control plane logic (~55KB)
  - `runtime-options.ts` - Runtime configuration

**ACP Bridge:** `openclaw acp` exposes an ACP agent over stdio and forwards prompts to the Gateway over WebSocket.

### 2.4 Plugin System (`src/plugins/` + `src/plugin-sdk/`)

OpenClaw uses a plugin architecture for extensibility. Plugins are workspace packages in `extensions/` that can provide:
- Channel integrations (Discord, Telegram, Slack, etc.)
- AI model providers (OpenAI, Anthropic, etc.)
- Skills and capabilities

**Plugin SDK (`src/plugin-sdk/`):**
- `core.ts` - Plugin entry points and helpers
- `index.ts` - Public API exports
- `runtime-api.ts` - Runtime API access
- Channel adapters and helpers

**Extension Structure:**
```
extensions/<name>/
├── package.json
├── openclaw.plugin.json  # Plugin manifest
├── src/
│   ├── api.ts           # Public API
│   ├── runtime-api.ts   # Runtime API
│   └── *.ts             # Implementation
└── ...
```

---

## 3. Channel System

### 3.1 Channel Architecture (`src/channels/`)

Channels are the inbound/outbound message processing components.

**Key Files:**
- `plugins/index.ts` - Channel plugin registry
- `plugins/types.plugin.ts` - Plugin type definitions
- `plugins/types.core.ts` - Core channel types
- `plugins/types.adapters.ts` - Adapter interfaces
- `channel-lifecycle.ts` - Channel lifecycle management
- `channel-config.ts` - Configuration handling

**Channel Plugin Types:**
- `ChannelPlugin` - Main plugin interface
- `ChannelMessagingAdapter` - Inbound/outbound messaging
- `ChannelOutboundAdapter` - Outbound message handling
- `ChannelPairingAdapter` - Device pairing
- `ChannelSecurityAdapter` - Security/allowlist handling

### 3.2 Built-in Channels (`src/`)

Core channel implementations in the main source:
- `telegram.ts` - Telegram support
- `signal.ts` - Signal messaging
- `slack.ts` - Slack integration
- `discord.ts` - Discord bot
- `imessage.ts` - iMessage via BlueBubbles
- `web.ts` - WebChat interface

### 3.3 Extension Channels (`extensions/`)

85 extension packages provide additional channel integrations:
- `discord/` - Discord bot
- `telegram/` - Telegram bot
- `slack/` - Slack integration
- `whatsapp/` - WhatsApp (Baileys-based)
- `signal/` - Signal
- `matrix/` - Matrix protocol
- `msteams/` - Microsoft Teams
- `line/` - LINE messaging
- And 77 more...

### 3.4 Channel Message Flow

```
Channel receives message
        │
        ▼
Channel Plugin normalizes message
        │
        ▼
Gateway routes to session
        │
        ▼
Pi Agent processes
        │
        ▼
Response flows back through
Gateway to Channel Plugin
        │
        ▼
Channel delivers response
```

---

## 4. Design Patterns

### 4.1 RPC Pattern

The Gateway uses a request/response RPC pattern over WebSocket:

**Request Frame:**
```typescript
{
  type: "req";
  id: string;
  method: string;
  params?: unknown;
}
```

**Response Frame:**
```typescript
{
  type: "res";
  id: string;
  ok: boolean;
  payload?: unknown;
  error?: { code: string; message: string; details?: unknown };
}
```

**Event Frame (server pushes):**
```typescript
{
  type: "event";
  event: string;
  payload?: unknown;
  seq?: number;
  stateVersion?: { presence: number; health: number };
}
```

### 4.2 Streaming Pattern

Tool streaming and block streaming use an iterative response pattern:
- Server sends partial updates as events
- Client receives `tool_call_update` events
- Final response sent when complete

### 4.3 Plugin SDK Pattern

Extensions use the Plugin SDK to interact with the Gateway:

```typescript
// Plugin entry
import { definePluginEntry } from "openclaw/plugin-sdk/core";
import { createChatChannelPlugin } from "openclaw/plugin-sdk/core";

export default definePluginEntry({
  channel: createChatChannelPlugin({
    // ... configuration
  }),
});
```

### 4.4 Channel Abstraction

All channel integrations implement a common interface:

```typescript
interface ChannelPlugin {
  id: string;
  name: string;
  capabilities: ChannelCapabilities;
  setup: ChannelSetupAdapter;
  send: ChannelSendAdapter;
  // ...
}
```

This allows the Gateway to treat all channels uniformly regardless of the underlying protocol.

---

## 5. Message Flow Architecture

### 5.1 Inbound Message Flow

```
External Message (Telegram/HTTP webhook/etc.)
        │
        ▼
Channel Plugin Entry Point
  (e.g., extensions/telegram/src/api.ts)
        │
        ▼
Gateway HTTP Handler
  (src/gateway/server-http.ts)
        │
        ▼
Hook Processing
  (src/gateway/hooks.ts)
        │
        ▼
Session Resolution
  (src/gateway/session-utils.ts)
        │
        ▼
Pi Agent Processing
  (src/agents/acp-spawn.ts)
        │
        ▼
Agent Event Stream
        │
        ├──► Tool Execution (Browser/Canvas/etc.)
        │
        └──► Response Generation
                │
                ▼
        Chat Event Handler
          (src/gateway/server-chat.ts)
                │
                ▼
        Outbound Channel Routing
                │
                ▼
        Channel Plugin Send
                │
                ▼
        External Platform Response
```

### 5.2 WebSocket Control Flow

```
UI / ACP Client
        │
        ▼
WebSocket Connection
  (src/gateway/server-ws-runtime.ts)
        │
        ▼
Connection Handler
  (src/gateway/server/ws-connection.ts)
        │
        ▼
Auth Handshake
        │
        ▼
RPC Method Dispatch
  (src/gateway/server-methods.ts)
        │
        ├──► chat.send
        ├──► sessions.list
        ├──► config.patch
        └──► ... (70+ methods)
                │
                ▼
        Response / Event Stream
```

### 5.3 UI to Gateway Connection

The Web UI (`ui/`) connects to the Gateway via WebSocket:

**Client Connection (`ui/src/ui/gateway.ts`):**
```typescript
// Connects to ws://127.0.0.1:18789
// Authenticates via token or device auth
// Sends RPC requests and receives events
```

**Key UI Components:**
- `app.ts` - Main application
- `app-chat.ts` - Chat interface
- `app-gateway.ts` - Gateway connection management
- `gateway.ts` - WebSocket client logic
- `views/` - View components (sidebar, chat, settings, etc.)

---

## 6. Session Management

### 6.1 Session Model (`src/gateway/session*.ts`)

Sessions are the core unit of conversation context:

**Session Key Format:**
```
agent:<agentId>:<sessionId>
main:acp:<uuid>     # ACP session
main:channel:discord:123456789  # Channel session
```

**Session Features:**
- Persistent across reconnects
- Transcript stored on disk
- Context compaction when limit reached
- Multiple sessions per agent
- Session modes (main, group isolation)

### 6.2 Session Lifecycle

```
Session Created
        │
        ▼
Messages Accumulate
        │
        ▼
Context Limit Reached
        │
        ▼
Compaction Triggered
  (src/context-engine/delegate.ts)
        │
        ▼
Summary + Key Facts Preserved
        │
        ▼
Session Continues
```

---

## 7. Security Architecture

### 7.1 Security Components (`src/security/`)

- `allowlists/` - Allowlist implementations
- `external-content.js` - External content handling
- `secret-equal.js` - Secret comparison

### 7.2 Authentication Methods

**Token-based:**
- Gateway auth token
- Bootstrap token
- Device token

**Pairing-based:**
- Device pairing flow
- DM pairing policy

### 7.3 Access Control

- `allowFrom` configuration per channel/account
- Per-session Docker sandboxing for non-main sessions
- Tool allowlist/denylist per session type
- Elevated bash access control

---

## 8. Skills Platform

### 8.1 Skills Architecture (`skills/`)

53 skill packages extend agent capabilities:

**Skill Categories:**
- **Bundled skills** - Core UX (acpx, acp transport)
- **Model providers** - AI integrations (anthropic, openai, etc.)
- **Channel integrations** - (discord, slack, etc.)
- **Media control** - (spotify-player, sonoscli)
- **Note-taking** - (notion, obsidian)
- **GitHub** - (github, gh-issues)
- **Content** - (summary, blogwatcher)

### 8.2 Skills Loading

Skills are discovered from:
- `~/.openclaw/workspace/skills/` - User-defined
- Bundled skills - Built into the system
- ClawHub (`clawhub.ai`) - Remote registry

---

## 9. Native Applications

### 9.1 macOS App (`apps/macos/`)

Swift application providing:
- Menu bar control plane
- Voice Wake + PTT overlay
- WebChat + debug tools
- Canvas, camera, screen recording
- `system.run`, `system.notify`
- Remote gateway control

### 9.2 iOS Node (`apps/ios/`)

Swift application with:
- Canvas surface
- Voice Wake + Talk Mode
- Camera snap/clip, screen recording
- Bonjour + device pairing

### 9.3 Android Node (`apps/android/`)

Kotlin application with:
- Connect tab (setup code/manual)
- Chat sessions, voice tab
- Canvas, camera/screen recording
- Device commands (notifications, location, SMS, photos, contacts, calendar)

### 9.4 Node Communication

Device nodes communicate with the Gateway via WebSocket using the same ACP protocol, advertising capabilities over the connection.

---

## 10. Key Technical Decisions

### 10.1 Framework Choice

- **Hono** for HTTP routing (fast, lightweight)
- **Express** compatibility layer for middleware
- **ws** for WebSocket server
- **TypeScript** throughout

### 10.2 Monorepo Structure

pnpm workspace with:
- Root package (CLI + core)
- `ui/` - Web UI package
- `packages/*` - Internal packages
- `extensions/*` - Plugin extensions (85 packages)
- `skills/` - Agent skills (53 packages)

### 10.3 Plugin Boundaries

Extension packages follow strict import boundaries:
- `openclaw/plugin-sdk/*` is the public contract
- Internal `src/**` imports prohibited
- Each extension has isolated `api.ts` / `runtime-api.ts` barrels

### 10.4 Session Isolation

- Main session has elevated privileges
- Non-main sessions run in Docker sandbox
- Tool access controlled per session type
- Separate transcript storage per session

---

## 11. Data Flow Summary

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           External World                                  │
│  Telegram │ Discord │ Slack │ WhatsApp │ Web │ ACP stdio │ Native Apps  │
└─────────────────┬───────────────────────────────────────────────────────┘
                  │
                  ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         Gateway Server                                     │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐    ┌─────────────┐ │
│  │   HTTP     │    │ WebSocket  │    │  Plugin     │    │    Cron     │ │
│  │  Handler   │    │  Handler   │    │   Host      │    │   Jobs      │ │
│  └─────┬──────┘    └─────┬──────┘    └──────┬─────┘    └──────┬──────┘ │
│        │                 │                   │                  │        │
│        └─────────────────┼───────────────────┼──────────────────┘        │
│                          │                                           │
│                    ┌─────▼─────┐                                     │
│                    │  Session   │                                     │
│                    │  Manager   │                                     │
│                    └─────┬─────┘                                     │
│                          │                                           │
│                    ┌─────▼─────┐                                     │
│                    │Pi Agent   │                                     │
│                    │Runtime    │                                     │
│                    └─────┬─────┘                                     │
│                          │                                           │
│        ┌─────────────────┼─────────────────┐                         │
│        │                 │                 │                         │
│  ┌─────▼─────┐     ┌─────▼─────┐     ┌─────▼─────┐                   │
│  │  Tools    │     │  Canvas   │     │  Browser  │                   │
│  │  (bash,   │     │   Host    │     │  Control  │                   │
│  │   etc.)   │     │   (A2UI)  │     │  (CDP)    │                   │
│  └───────────┘     └───────────┘     └───────────┘                   │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 12. Configuration

### 12.1 Config Location

- `~/.openclaw/openclaw.json` - User configuration
- `~/.openclaw/sessions/` - Session data
- `~/.openclaw/credentials/` - Stored credentials
- `~/.openclaw/workspace/skills/` - User skills

### 12.2 Config Schema

Generated via `pnpm config:schema:gen` and validated with Zod schemas at runtime.

---

## 13. Dependencies

### Core Runtime
- `hono` (4.12.8) - Web framework
- `ws` (8.20.0) - WebSocket
- `zod` (4.3.6) - Schema validation
- `@sinclair/typebox` (0.34.48) - Type schemas

### AI/Agent
- `@mariozechner/pi-agent-core` (0.61.1)
- `@mariozechner/pi-ai` (0.61.1)
- `@modelcontextprotocol/sdk` (1.27.1)

### Build
- TypeScript 5.9.3
- tsdown for bundling
- pnpm 10.32.1

---

## 14. Testing Infrastructure

**Vitest Configuration:**
- `vitest.config.ts` - Default config
- `vitest.unit.config.ts` - Unit tests
- `vitest.e2e.config.ts` - E2E tests
- `vitest.gateway.config.ts` - Gateway tests
- `vitest.live.config.ts` - Live/integration tests

**Test Fixtures:**
- `test/fixtures/` - Test fixtures
- `test/helpers/` - Test helpers
- `test/mocks/` - Mock implementations

---

## 15. Summary

OpenClaw's architecture is built around a **hub-and-spoke model**:

1. **Gateway** is the central hub - a Hono/Express server on port 18789
2. **Channels** are spokes - plugins that normalize diverse messaging protocols
3. **Pi Agent** is the brain - processes messages and generates responses
4. **Sessions** provide context - persistent conversation state per user/conversation
5. **Skills** extend capabilities - modular additions to agent functionality
6. **Native apps** are companions - iOS/Android/macOS that connect via ACP

The design prioritizes:
- **Local-first** - All processing happens locally
- **Extensibility** - Plugin architecture for channels, providers, skills
- **Security** - Docker sandboxing, allowlists, pairing-based access
- **Multi-channel** - Unified inbox across 20+ messaging platforms
