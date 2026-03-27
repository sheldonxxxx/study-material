# OpenClaw Architecture

**Study Note:** Synthesized from research/01-topology.md, 04-features-index.md, 06-architecture.md
**Project:** OpenClaw - Personal AI Assistant
**Date:** 2026-03-26

---

## 1. Architectural Pattern: Hub-and-Spoke

OpenClaw follows a **hub-and-spoke architecture** where the Gateway serves as the central hub connecting all messaging channels (spokes) to the AI agent runtime.

```
                         Messaging Channels
    WhatsApp | Telegram | Slack | Discord | Signal | iMessage | Matrix | ...
                                   |
                                   v
    +-------------------------------------------------------------------+
    |                        OpenClaw Gateway                           |
    |              (Hono + Express + WebSocket :18789)                  |
    |  +-----------+ +-----------+ +-----------+ +-------------------+ |
    |  | Sessions | | Channels  | |   Tools   | | Cron/Webhooks/MCP| |
    |  |           | |           | | Browser   | |                   | |
    |  |           | |           | | Canvas    | |                   | |
    |  +-----+-----+ +-----+-----+ +-----+-----+ +---------+---------+ |
    |        +----------------+-----------------+                   |
    |                         |                                      |
    |                  +-------v--------+                             |
    |                  |  Pi Agent      |                             |
    |                  |  (Runtime)     |                             |
    |                  +----------------+                             |
    +-------------------------------+---------------------------------+
                                    |
          +-------------------------+-------------------------+
          |                         |                         |
          v                         v                         v
    +-----------+            +-----------+            +-----------+
    |  macOS    |            |   iOS     |            |  Android  |
    |  Menu bar |            |   Node    |            |   Node    |
    |  Voice    |            |  Canvas   |            |  Chat/    |
    |  WebChat  |            |  Voice    |            |  Voice    |
    +-----------+            +-----------+            +-----------+
```

---

## 2. Core Components

### 2.1 Gateway (`src/gateway/`)

The **Gateway** is the central server that orchestrates all components.

**Technology Stack:**
- Hono (HTTP routing) + Express compatibility
- WebSocket server (ws library)
- Default port: `18789` (localhost)

**Key Files:**
| File | Purpose |
|------|---------|
| `server.impl.ts` | Main server implementation (1400+ lines) |
| `server-http.ts` | HTTP request handling, routes, hooks, auth |
| `server-ws-runtime.ts` | WebSocket runtime setup |
| `server-chat.ts` | Chat session handling |
| `server-methods.ts` | RPC method handlers (70+ methods) |

**Responsibilities:**
- Session management with presence and typing indicators
- Channel coordination and message routing
- Tool execution coordination (browser, canvas, etc.)
- Cron jobs and webhook triggers
- Authentication and access control
- Configuration management

---

### 2.2 Pi Agent Runtime (`src/agents/`)

The **Pi Agent** is the core AI processing engine that processes messages, executes tools, and coordinates responses.

**Key Components:**
| File | Purpose |
|------|---------|
| `acp-spawn.ts` | Agent spawning logic |
| `agent-command.ts` | Command handling (~43KB) |
| `agent-scope.ts` | Session scoping |
| `pi-embedded-runner/` | Embedded agent execution |

**Agent Features:**
- RPC-based agent runtime
- Tool streaming and block streaming
- Multi-agent routing (route channels/accounts to isolated agents)
- Session model: main for direct chats, group isolation
- Context compaction and session pruning
- Reply-back and queue modes

---

### 2.3 ACP Protocol (`src/acp/`)

The **Agent Client Protocol (ACP)** is the wire protocol for IDE-to-gateway communication.

**Key Files:**
- `session.ts` - Session management
- `client.ts` - ACP client implementation
- `control-plane/manager.core.ts` - Core control plane logic (~55KB)
- `control-plane/runtime-options.ts` - Runtime configuration

**ACP Bridge:** `openclaw acp` exposes an ACP agent over stdio and forwards prompts to the Gateway over WebSocket.

---

### 2.4 Plugin System (`src/plugins/` + `src/plugin-sdk/`)

OpenClaw uses a **plugin architecture** for extensibility. Plugins provide:
- Channel integrations (Discord, Telegram, Slack, etc.)
- AI model providers (OpenAI, Anthropic, etc.)
- Skills and capabilities

**Plugin SDK (`src/plugin-sdk/`):**
- `core.ts` - Plugin entry points and helpers
- `index.ts` - Public API exports
- `runtime-api.ts` - Runtime API access

**Extension Structure:**
```
extensions/<name>/
├── package.json
├── openclaw.plugin.json  # Plugin manifest
├── src/
│   ├── api.ts           # Public API (extension -> openclaw)
│   ├── runtime-api.ts   # Runtime API
│   └── *.ts             # Implementation
```

**85 extension packages** provide additional integrations.

---

### 2.5 Channel System (`src/channels/`)

Channels are the **inbound/outbound message processing components** that normalize diverse messaging protocols.

**Key Types:**
- `ChannelPlugin` - Main plugin interface
- `ChannelMessagingAdapter` - Inbound/outbound messaging
- `ChannelOutboundAdapter` - Outbound message handling
- `ChannelPairingAdapter` - Device pairing
- `ChannelSecurityAdapter` - Security/allowlist handling

**Channel Plugin Types:**
| Type | Purpose |
|------|---------|
| Built-in | Core implementations: `telegram.ts`, `signal.ts`, `slack.ts`, `discord.ts`, `imessage.ts`, `web.ts` |
| Extensions | 85 packages: `discord/`, `telegram/`, `slack/`, `whatsapp/`, `matrix/`, `msteams/`, `line/`, and 77 more |

---

## 3. Message Flow Through the System

### 3.1 Inbound Message Flow

```
External Message (Telegram/HTTP webhook/etc.)
        |
        v
Channel Plugin Entry Point
  (e.g., extensions/telegram/src/api.ts)
        |
        v
Gateway HTTP Handler
  (src/gateway/server-http.ts)
        |
        v
Hook Processing
  (src/gateway/hooks.ts)
        |
        v
Session Resolution
  (src/gateway/session-utils.ts)
        |
        v
Pi Agent Processing
  (src/agents/acp-spawn.ts)
        |
        v
Agent Event Stream
        |
        +---> Tool Execution (Browser/Canvas/etc.)
        |
        +---> Response Generation
                |
                v
        Chat Event Handler
          (src/gateway/server-chat.ts)
                |
                v
        Outbound Channel Routing
                |
                v
        Channel Plugin Send
                |
                v
        External Platform Response
```

### 3.2 WebSocket Control Flow

```
UI / ACP Client
        |
        v
WebSocket Connection
  (src/gateway/server-ws-runtime.ts)
        |
        v
Connection Handler
  (src/gateway/server/ws-connection.ts)
        |
        v
Auth Handshake
        |
        v
RPC Method Dispatch
  (src/gateway/server-methods.ts)
        |
        +---> chat.send
        +---> sessions.list
        +---> config.patch
        +---> ... (70+ methods)
                |
                v
        Response / Event Stream
```

---

## 4. Design Patterns

### 4.1 RPC over WebSocket

The Gateway uses a **request/response RPC pattern** over WebSocket:

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

### 4.2 Plugin SDK Pattern

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

### 4.3 Channel Abstraction

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

### 4.4 Streaming Pattern

Tool streaming and block streaming use an **iterative response pattern**:
- Server sends partial updates as events
- Client receives `tool_call_update` events
- Final response sent when complete

---

## 5. Module Responsibilities

| Module | Responsibility | Location |
|--------|---------------|----------|
| Gateway | Central server, routing, session management | `src/gateway/` |
| Pi Agent | AI processing, tool execution | `src/agents/` |
| ACP Protocol | Wire protocol for IDE-to-gateway | `src/acp/` |
| Plugin SDK | Extension development interface | `src/plugin-sdk/` |
| Channels | Message normalization | `src/channels/` |
| Browser | CDP-controlled Chrome automation | `src/browser/` |
| Canvas | Agent-to-UI visual workspace | `src/canvas-host/` |
| Cron | Scheduled automation | `src/cron/` |
| Security | Allowlists, sandboxing, secrets | `src/security/` |
| Media | Image/audio/video processing | `src/media/` |

---

## 6. ACP Protocol Overview

**ACP (Agent Client Protocol)** is the wire protocol for IDE-to-gateway communication.

**Key Concepts:**
- Session-based communication
- Request/response RPC over WebSocket
- Event streaming for tool calls and block updates
- ACP Bridge: `openclaw acp` exposes an ACP agent over stdio

**Session Key Format:**
```
agent:<agentId>:<sessionId>
main:acp:<uuid>     # ACP session
main:channel:discord:123456789  # Channel session
```

---

## 7. Session Management

**Session Model:**
- Sessions are the core unit of conversation context
- Persistent across reconnects
- Transcript stored on disk
- Context compaction when limit reached
- Multiple sessions per agent
- Session modes (main, group isolation)

**Session Lifecycle:**
```
Session Created
        |
        v
Messages Accumulate
        |
        v
Context Limit Reached --> Compaction Triggered
        |                    |
        v                    v
Session Continues    Summary + Key Facts Preserved
```

---

## 8. Security Architecture

**Security Components (`src/security/`):**
- `allowlists/` - Allowlist implementations
- `external-content.js` - External content handling
- `secret-equal.js` - Secret comparison

**Authentication Methods:**
- Token-based (Gateway auth, Bootstrap, Device token)
- Pairing-based (Device pairing flow, DM pairing policy)

**Access Control:**
- `allowFrom` configuration per channel/account
- Per-session Docker sandboxing for non-main sessions
- Tool allowlist/denylist per session type
- Elevated bash access control

---

## 9. Key Architectural Decisions

| Decision | Rationale |
|----------|-----------|
| Hub-and-spoke | Central gateway unifies all channels |
| Hono + Express + ws | Fast HTTP routing with Express compatibility and WebSocket |
| RPC over WebSocket | Bi-directional, real-time communication |
| Plugin SDK boundary | Extensions must use `openclaw/plugin-sdk/*` imports only |
| Session isolation | Main session elevated, others sandboxed |
| Local-first | All processing happens locally |
| ESM-only | Strict ESM throughout, no CommonJS |

---

## Summary

OpenClaw's architecture is built around a **hub-and-spoke model**:

1. **Gateway** - Central hub on port 18789 (Hono/Express/WebSocket)
2. **Channels** - Spokes that normalize diverse messaging protocols
3. **Pi Agent** - The brain that processes messages and generates responses
4. **Sessions** - Persistent conversation state per user/conversation
5. **Skills** - Modular capability additions to the agent
6. **Native apps** - iOS/Android/macOS companions that connect via ACP

The design prioritizes **local-first**, **extensibility**, **security**, and **multi-channel** unification.
