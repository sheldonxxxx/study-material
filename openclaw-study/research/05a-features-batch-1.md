# Feature Batch 1 Deep Dive: Multi-Channel Messaging Hub, Local-First Gateway Control Plane, AI Agent Runtime (Pi Agent)

**Date:** 2026-03-26
**Repo:** `/Users/sheldon/Documents/claw/reference/openclaw`
**Source:** Feature index (`04-features-index.md`), Topology (`01-topology.md`), and direct source exploration

---

## Feature 1: Multi-Channel Messaging Hub

### Overview

The Multi-Channel Messaging Hub connects 20+ messaging platforms through a unified plugin architecture. The core abstraction is the `ChannelPlugin` interface, with plugins residing in `extensions/` as independent packages.

### Channel Plugin Architecture

#### Plugin Registration System

Plugins register via `defineChannelPluginEntry()` in `src/plugin-sdk/core.ts`:

```typescript
export function defineChannelPluginEntry<TPlugin>({
  id, name, description, plugin, configSchema, setRuntime, registerFull
}) {
  return definePluginEntry({
    id, name, description, configSchema,
    register(api: OpenClawPluginApi) {
      setRuntime?.(api.runtime);
      api.registerChannel({ plugin: plugin as ChannelPlugin });
      if (api.registrationMode !== "full") return;
      registerFull?.(api);
    },
  });
}
```

The registry (`src/plugins/registry.ts`) maintains separate `channels` and `channelSetups` arrays:
- `channelSetups`: All registered plugins (for setup wizard rendering)
- `channels`: Fully activated plugins (for runtime operations)

#### Channel Abstraction (`createChatChannelPlugin`)

All chat channels (Discord, Telegram, Slack, WhatsApp) use `createChatChannelPlugin()` which composes:

1. **base**: `id`, `meta`, `setup`, `setupWizard`, `capabilities`
2. **security**: DM policy (`resolveDmPolicy`, `resolveAllowFrom`), group warnings
3. **pairing**: Text-based pairing with `idLabel`, `message`, `notify()`
4. **threading**: `topLevelReplyToMode` per channel
5. **outbound**: `deliveryMode`, `chunker`, `sendPayload`, `sendText`, `sendMedia`, `sendPoll`

#### Example: Discord Plugin Structure (`extensions/discord/src/channel.ts`)

The Discord plugin is 700+ lines defining:

- **Account resolution**: `resolveDiscordAccount()` retrieves config for each account
- **Session routing**: `resolveDiscordOutboundSessionRoute()` builds session keys like `discord:channel:123` or `discord:user:456`
- **Messaging**: `normalizeDiscordMessagingTarget()` parses targets like `channel:123`, `user:@username`
- **Permissions**: `auditDiscordChannelPermissions()` checks bot permissions in channels
- **Actions**: `discordMessageActions` handles `timeout`, `kick`, `ban` with trusted requester verification
- **Pairing**: Text-based pairing with `PAIRING_APPROVED_MESSAGE`

#### Key Channel Patterns

**Target normalization** (`extensions/*/src/normalize.ts`):
```typescript
// Discord
looksLikeDiscordTargetId(target: string): boolean // /\d+/

// Telegram
parseTelegramTarget(raw: string): { chatId, chatType, messageThreadId }

// WhatsApp
normalizeWhatsAppTarget(raw: string): string | null // E.164 or JID
```

**Outbound session routing** builds a `ChannelOutboundSessionRoute`:
```typescript
{
  sessionKey: "agent:main:acp:uuid",    // Pi agent session key
  baseSessionKey: "...",                // Without thread context
  peer: { kind: "direct" | "group" | "channel", id: "..." },
  chatType: "direct" | "group" | "channel",
  from: "telegram:12345",
  to: "telegram:67890",
  threadId: "..."
}
```

**Account configuration** (`channels/plugins/types.plugin.ts`):
```typescript
interface ChannelPlugin<TResolvedAccount, Probe = unknown, Audit = unknown> {
  id: string;
  meta?: { aliases?: string[]; label?: string; ... };
  capabilities?: { streaming?: boolean; };
  setup: { surface: ..., account: ..., configure: ... };
  gateway?: {
    startAccount(ctx: ChannelGatewayContext): Promise<ChannelAccountRuntime>;
    stopAccount?(ctx: ChannelGatewayContext): Promise<void>;
  };
  status?: {
    probeAccount(account, timeoutMs): Promise<Probe>;
    resolveAccountSnapshot(account, runtime, probe, audit): ChannelAccountSnapshot;
  };
  ...
}
```

### Channel Registry

The channel registry (`src/channels/registry.ts`) provides:

```typescript
// Built-in chat channels (hardcoded metadata)
const CHAT_CHANNEL_META: Record<ChatChannelId, ChannelMeta> = {
  telegram: { id, label, selectionLabel, docsPath, systemImage, ... },
  whatsapp: { ... },
  discord: { ... },
  ...
}

// Normalization (for lookup)
normalizeChatChannelId(raw?: string | null): ChatChannelId | null
normalizeAnyChannelId(raw?: string | null): ChannelId | null // Checks plugin registry

// Listing
listChatChannels(): ChatChannelMeta[]
listRegisteredChannelPluginIds(): ChannelId[]
```

### Clever Solutions

1. **Lazy runtime loading**: Discord provider uses `loadDiscordProviderRuntime()` with promise caching to avoid eager loading heavy Discord.js dependency
2. **Channel-specific text chunking**: Telegram uses `chunkMarkdownText()` with 4000 char limit; Discord uses plain 2000 char chunks
3. **ACP conversation binding**: Each channel implements `compileConfiguredBinding` and `matchInboundConversation` for thread binding
4. **Dual registration mode**: `setup-only` (wizard rendering) vs `full` (runtime activation)

### Technical Debt / Concerns

1. **Massive channel.ts files**: Discord channel.ts is 700+ lines, Telegram is 800+ lines - monolithic files
2. **Inconsistent patterns**: WhatsApp uses `runtime-api.js` for shared exports; Discord has its own `runtime-api.ts`
3. **Plugin boundary violations**: Extensions import from `openclaw/plugin-sdk/*` but some internal modules are imported directly in implementation
4. **WhatsApp Web vs Business API**: WhatsApp implementation is Web-based (QR code login) - fragile and requires persistent browser session

---

## Feature 2: Local-First Gateway Control Plane

### Overview

The Gateway is a WebSocket-based control plane running on `ws://127.0.0.1:18789` by default. It serves as the central hub for sessions, channels, tools, cron jobs, and device nodes.

### Architecture

#### Server Stack

```
server-http.ts       - HTTP server (Hono/Express)
server-ws-runtime.ts  - WebSocket upgrade handling
server.impl.ts       - Main gateway implementation
server-methods.ts    - Core gateway methods
server-plugins.ts    - Plugin integration
```

#### Key Components

**Channel Manager** (`server-channels.ts`):
```typescript
createChannelManager({
  loadConfig,           // Config getter
  channelLogs,          // Per-channel loggers
  channelRuntimeEnvs,   // Runtime environments
  channelRuntime?,       // Optional Plugin SDK channel runtime
  resolveChannelRuntime? // Lazy resolver
}) => ChannelManager

// Interface
interface ChannelManager {
  getRuntimeSnapshot(): ChannelRuntimeSnapshot;
  startChannels(): Promise<void>;
  startChannel(channel: ChannelId, accountId?: string): Promise<void>;
  stopChannel(channel: ChannelId, accountId?: string): Promise<void>;
  markChannelLoggedOut(channelId, cleared, accountId?): void;
  isManuallyStopped(channelId, accountId): boolean;
}
```

**Session Management** (`config/sessions/`):
- `store.ts`: Session storage with `loadSessionStore()`, `updateSessionStore()`
- `main-session.ts`: Resolves main session keys like `agent:main:acp:uuid`
- `transcript.ts`: Session transcript file management

**Gateway Methods** (exposed via WebSocket/HTTP):
```typescript
// From server-methods.ts
const coreGatewayHandlers = {
  "sessions.patch": handleSessionsPatch,
  "sessions.send": handleSessionsSend,
  "sessions.history": handleSessionsHistory,
  "agent": handleAgentMessage,
  "tools.invoke": handleToolsInvoke,
  "channels.start": handleChannelsStart,
  "channels.stop": handleChannelsStop,
  ...
}
```

### Boot Process

`boot.ts` implements a startup check system:

```typescript
runBootOnce({
  cfg, deps, workspaceDir, agentId?
}): Promise<BootRunResult>

// Reads ~/.openclaw/workspace/BOOT.md
// Prompts Pi agent with BOOT.md content
// Uses SILENT_REPLY_TOKEN to suppress output if no action needed
```

The boot session uses a generated ID (`boot-{timestamp}-{uuid}`) and snapshots/restores main session mappings to avoid side effects.

### Session Lifecycle

**Session states** (`session-lifecycle-state.ts`):
```typescript
type SessionLifecycleState =
  | "pending"
  | "input"
  | "thinking"
  | "streaming"
  | "tool_use"
  | "completed"
  | "error";
```

**Session key structure**:
```
agent:{agentId}:{mode}:{sessionId}
agent:main:acp:{uuid}           // Main ACP session
agent:{agentId}:cli:{uuid}      // CLI session
```

### WebSocket Control Protocol

**Client** (`client.ts`):
- `connectWebSocket(url, token)`: Creates WebSocket connection
- `attachWsHandlers(handlers)`: Sets up message handlers
- `watchdog`: Detects connection failures

**Server** (`server-ws-runtime.ts`):
- Validates auth token on connection
- Routes messages to handlers by method name
- Broadcasts events (agent events, typing indicators)

### Health & Monitoring

**Health endpoints** (`/health`, `/ready`):
- `/health` - liveness probe
- `/ready` - readiness probe

**Channel health monitor** (`channel-health-monitor.ts`):
```typescript
startChannelHealthMonitor({
  checkInterval,
  timeout,
  onChannelDown(channel, accountId, error),
  onChannelUp(channel, accountId)
})
```

### Technical Debt / Concerns

1. **Server monolithic**: `server.impl.ts` is 2000+ lines handling initialization
2. **Config hot-reload complexity**: `config-reload.ts` and `server-reload-handlers.ts` show tension between mutable config and clean reload
3. **Session store race conditions**: Multiple write paths to session store with potential race conditions
4. **Memory pressure**: Sessions store full transcripts; no automatic archival in gateway

---

## Feature 3: AI Agent Runtime (Pi Agent)

### Overview

The Pi Agent is an RPC-based runtime that processes messages, executes tools, and coordinates responses. It uses `@mariozechner/pi-agent-core` and `@mariozechner/pi-ai` as underlying engines.

### Agent Spawning

**ACP Spawn** (`agents/acp-spawn.ts`):

```typescript
spawnAcpDirect(
  params: {
    task: string;           // Task description
    label?: string;         // Human-readable label
    agentId?: string;      // Target agent (defaults to config)
    resumeSessionId?: string;
    cwd?: string;
    mode?: "run" | "session";  // Oneshot vs persistent
    thread?: boolean;       // Thread-bound spawn
    sandbox?: "inherit" | "require";
    streamTo?: "parent";   // Stream output to parent
  },
  ctx: SpawnAcpContext     // Requester context
): Promise<SpawnAcpResult>
```

**Spawn modes**:
- `run`: One-shot execution, result returned inline
- `session`: Persistent session with thread binding

**Session binding** for ACP sessions:
- `prepareAcpThreadBinding()`: Validates channel supports thread binding
- `bindPreparedAcpThread()`: Creates `SessionBindingRecord` linking child session to channel thread
- Heartbeat relay: `startAcpSpawnParentStreamRelay()` for streaming output to parent

### Tool Streaming & Block Streaming

**Block Reply Pipeline** (`auto-reply/reply/block-reply-pipeline.ts`):

```typescript
createBlockReplyPipeline({
  onBlockReply(payload, { abortSignal?, timeoutMs? }): Promise<void>;
  timeoutMs: number;
  coalescing?: BlockStreamingCoalescing;
  buffer?: BlockReplyBuffer;
}) => BlockReplyPipeline

// Pipeline stages:
// 1. buffer (optional) - buffers until condition met
// 2. coalesce - batches text chunks to reduce network calls
// 3. send - delivers to channel
```

**Coalescing configuration**:
```typescript
{
  maxChars: 200,           // Flush when buffered text exceeds
  minChars: 50,            // Minimum before flush
  idleMs: 1000,            // Flush after idle period
  joiner: "\n\n",          // Text joiner
  flushOnParagraph: true   // Flush at paragraph boundary
}
```

**Block streaming** (`auto-reply/reply/block-streaming.ts`):
- `resolveBlockStreamingChunking()`: Per-channel chunk size limits
- `resolveEffectiveBlockStreamingConfig()`: ACP overrides, coalescing bounds

### Context Engine

**Registry pattern** (`context-engine/registry.ts`):

```typescript
registerContextEngineForOwner(
  id: string,
  factory: ContextEngineFactory, // () => ContextEngine | Promise<ContextEngine>
  owner: string,
  opts?: { allowSameOwnerRefresh?: boolean }
): ContextEngineRegistrationResult

// Module-level singleton registry
// Legacy sessionKey compat with proxy wrapping
```

**Resolution**:
```typescript
resolveContextEngine(config?): ContextEngine
// 1. Check config.plugins.slots.contextEngine
// 2. Fall back to default slot ("legacy")
// 3. Wrap with sessionKey compat proxy
```

### Agent Command Processing

**`agent-command.ts`** (43KB, 900+ lines):

Key flow:
1. **Ingress**: `agentCommand()` receives message + session key
2. **Session resolution**: `resolveSession()` loads/creates session entry
3. **Model selection**: `resolveConfiguredModelRef()`, `resolveDefaultModelForAgent()`
4. **Auth profiles**: `ensureAuthProfileStore()` for API key rotation
5. **Embedded runner**: `runEmbeddedPiAgent()` executes with tools
6. **Response delivery**: `deliverAgentCommandResult()` sends to channel

**ACP visible text accumulator**:
```typescript
createAcpVisibleTextAccumulator()
// Handles silent reply tokens, cumulative snapshots
// Merges chunks while respecting silent prefix
```

### Session Isolation

**Lane system** (`agents/lanes.js`):
```typescript
AGENT_LANE_SUBAGENT = "subagent"
```
Sessions can be tagged with lanes for concurrency control and isolation.

**Sandbox runtime** (`agents/sandbox/runtime-status.ts`):
```typescript
resolveSandboxRuntimeStatus({ cfg, sessionKey })
// "sandboxed" | "host" | "unknown"
```

### Multi-Agent Routing

**Agent scoping** (`agents/agent-scope.ts`):
```typescript
resolveAgentDir(cfg, agentId): string     // ~/.openclaw/agents/{agentId}
resolveAgentWorkspaceDir(cfg, agentId)    // Agent-specific workspace
resolveDefaultAgentId(cfg): string        // Configured default
```

**Subagent registry** (`agents/subagent-registry.ts`):
- Tracks spawned subagents per parent session
- Handles cleanup on parent session end

### Clever Solutions

1. **Idempotency keys**: ACP spawn uses `crypto.randomUUID()` as idempotency key to prevent duplicate runs
2. **Session snapshot/restore**: Boot process snapshots main session mapping, runs agent, restores - avoids side effects
3. **ACP stream relay**: Fast failures caught before relay registration; gateway returns `runId` that updates relay
4. **Legacy compat proxy**: Context engine registry wraps engines with sessionKey compat proxy that auto-detects API shape

### Technical Debt / Concerns

1. **Massive files**: `agent-command.ts` is 43KB+ monolithic, hard to navigate
2. **ACP spawn complexity**: 900+ line function with deep nested logic for thread binding, stream relay, delivery planning
3. **Session store mutations**: `mergeSessionEntry()` with `OVERRIDE_FIELDS_CLEARED_BY_DELETE` is fragile
4. **Auth profile leakage**: Session-level auth profile overrides persist across sessions if not cleared
5. **No context compaction visible**: Despite mentioning "context compaction" in feature description, implementation not explored in depth

---

## Cross-Feature Integration

### Message Flow

```
Channel Plugin (e.g., Telegram)
  |
  v
gateway/server-http.ts (HTTP webhook) / server-ws-runtime.ts (WebSocket)
  |
  v
server-chat.ts (message handling)
  |
  v
agent-command.ts (Pi agent processing)
  |
  v
Tool execution (via tools-invoke-http.ts)
  |
  v
Block reply pipeline (coalescing, chunking)
  |
  v
Channel outbound adapter (sendMessageTelegram, etc.)
  |
  v
Channel Plugin (delivery to user)
```

### Session Key Routing

```
telegram:12345 (channel + target)
  |
  v
resolveTelegramOutboundSessionRoute()
  |
  v
agent:main:acp:{uuid} (Pi session)
  |
  v
ACP spawn (optional child session)
  |
  v
Thread binding (for persistent sessions)
```

---

## Key Files Reference

### Multi-Channel Messaging
| File | Purpose |
|------|---------|
| `src/channels/registry.ts` | Channel metadata, normalization |
| `src/plugin-sdk/core.ts` | `createChatChannelPlugin()`, `defineChannelPluginEntry()` |
| `src/plugins/registry.ts` | Plugin registry, `registerChannel()` |
| `extensions/discord/src/channel.ts` | Discord plugin (700+ lines) |
| `extensions/telegram/src/channel.ts` | Telegram plugin (800+ lines) |
| `extensions/whatsapp/src/channel.ts` | WhatsApp plugin (400+ lines) |

### Gateway Control Plane
| File | Purpose |
|------|---------|
| `src/gateway/server-http.ts` | HTTP server with webhook handlers |
| `src/gateway/server.impl.ts` | Main gateway implementation |
| `src/gateway/server-channels.ts` | Channel lifecycle manager |
| `src/gateway/boot.ts` | Startup boot check system |
| `src/gateway/server-ws-runtime.ts` | WebSocket handling |

### AI Agent Runtime
| File | Purpose |
|------|---------|
| `src/agents/acp-spawn.ts` | ACP session spawning (900+ lines) |
| `src/agents/agent-command.ts` | Agent message processing (43KB) |
| `src/context-engine/registry.ts` | Context engine factory registry |
| `src/auto-reply/reply/block-reply-pipeline.ts` | Streaming pipeline with coalescing |
| `src/auto-reply/reply/block-streaming.ts` | Chunking configuration |

---

## Summary

The OpenClaw architecture shows a well-designed plugin system for channels with clear separation between setup (wizard) and runtime modes. The Gateway is a capable control plane but exhibits monolith tendencies in core files. The Pi Agent runtime is sophisticated with ACP spawning, block streaming, and multi-agent support, but the implementation complexity (especially `acp-spawn.ts`) suggests technical debt.

**Strengths:**
- Clean plugin API via `defineChannelPluginEntry()` and `createChatChannelPlugin()`
- Sophisticated block streaming with coalescing to reduce chat spam
- Session binding for persistent agent threads
- Lazy loading patterns for heavy dependencies

**Weaknesses:**
- Monolithic channel.ts files (700-800+ lines)
- Complex ACP spawn logic with deep nesting
- Session store mutation patterns that are fragile
- WhatsApp Web-based implementation being inherently fragile
