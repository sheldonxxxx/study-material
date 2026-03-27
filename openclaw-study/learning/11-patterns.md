# OpenClaw Design & Architecture Patterns

**Study Root:** `/Users/sheldon/Documents/claw/openclaw-study`
**Date:** 2026-03-26
**Source:** Analysis of all research documents (01-08)

---

## 1. Plugin SDK Pattern

### `definePluginEntry` / `defineChannelPluginEntry`

Extensions register via a unified entry point that handles lifecycle:

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

**Key Pattern:** Dual registration mode (`setup-only` vs `full`) allows plugins to render in setup wizards without activating runtime.

### `createChatChannelPlugin` Composition

All chat channels use a factory that composes common concerns:

```typescript
createChatChannelPlugin({
  base: { id, meta, setup, setupWizard, capabilities },
  security: { resolveDmPolicy, resolveAllowFrom },
  pairing: { idLabel, message, notify },
  threading: { topLevelReplyToMode },
  outbound: { deliveryMode, chunker, sendPayload, sendText, sendMedia, sendPoll }
});
```

**Benefit:** Shared patterns (DM policy, pairing, threading) codified once, applied consistently across 20+ channels.

---

## 2. Channel Abstraction Pattern

### Unified Interface, Platform-Specific Implementation

Each channel plugin implements a common interface but normalizes platform-specific quirks internally:

| Concern | Pattern |
|---------|---------|
| Target normalization | `parseTelegramTarget()`, `normalizeWhatsAppTarget()` |
| Session routing | `resolveDiscordOutboundSessionRoute()` builds `ChannelOutboundSessionRoute` |
| Account resolution | `resolveDiscordAccount()` retrieves config per account |
| Permissions | `auditDiscordChannelPermissions()` checks bot permissions |

### Registry with Separation

```typescript
// channelSetups: All registered plugins (for setup wizard)
// channels: Fully activated plugins (for runtime)
const registry = { channelSetups: [], channels: [] };
```

---

## 3. ACP RPC over WebSocket

### Request/Response Pattern

```typescript
// Request Frame
{ type: "req", id: string, method: string, params?: unknown }

// Response Frame
{ type: "res", id: string, ok: boolean, payload?: unknown, error?: { code, message } }

// Event Frame (server pushes)
{ type: "event", event: string, payload?: unknown, seq?: number }
```

### Idempotency via UUIDs

> "Idempotency keys are required for side-effecting methods (`send`, `agent`) to safely retry; the server keeps a short-lived dedupe cache."

ACP spawn uses `crypto.randomUUID()` as idempotency key to prevent duplicate runs.

---

## 4. Block Streaming with Coalescing

### Pipeline Architecture

```typescript
createBlockReplyPipeline({
  onBlockReply(payload, { abortSignal?, timeoutMs? }),
  timeoutMs: number,
  coalescing?: BlockStreamingCoalescing,
  buffer?: BlockReplyBuffer
});

// Pipeline stages: buffer -> coalesce -> send
```

### Coalescing Configuration

```typescript
{
  maxChars: 200,        // Flush when buffered text exceeds
  minChars: 50,         // Minimum before flush
  idleMs: 1000,         // Flush after idle period
  joiner: "\n\n",       // Text joiner
  flushOnParagraph: true // Flush at paragraph boundary
}
```

**Benefit:** Reduces chat spam by batching text chunks; respects platform message limits per-channel.

---

## 5. Session Isolation (Lane System + Sandbox Runtime)

### Lane Tags

```typescript
AGENT_LANE_SUBAGENT = "subagent"
// Sessions can be tagged for concurrency control and isolation
```

### Sandbox Runtime Resolution

```typescript
resolveSandboxRuntimeStatus({ cfg, sessionKey })
// Returns: "sandboxed" | "host" | "unknown"
```

### Docker Sandboxing

```typescript
resolveSandboxConfigForAgent({
  scope: "agent" | "session" | "shared",
  // agent-specific overrides merged with global config
});
```

- Main session runs on host (elevated privileges)
- Non-main sessions run in Docker sandbox by default
- Tool access controlled per session type via allowlists

---

## 6. Lazy Loading with Promise Caching

### Module-Level Promise Cache

```typescript
// From src/security/audit.ts
channelPluginsModulePromise ??= import("../channels/plugins/index.js");
// Module-level promise caching prevents duplicate dynamic imports
```

### Runtime Lazy Loading

Discord provider uses `loadDiscordProviderRuntime()` with promise caching to avoid eager loading heavy Discord.js dependency.

**Pattern:** Store promise in module-level variable, check null before calling `import()`.

---

## 7. External Content Wrapping (Injection Prevention)

### Unique Boundary Markers

```typescript
createExternalContentMarkerId() -> randomBytes(8).toString("hex")
```

### Suspicious Pattern Detection

```typescript
SUSPICIOUS_PATTERNS = [
  "ignore previous instructions",
  "forget everything",
  "new instructions:",
  "system prompt override",
  "exec/rm -rf commands",
  "<system> tags"
];
```

**Content sources:** email, webhook, api, browser, channel_metadata, web_search, web_fetch

---

## 8. Context Engine Registry Pattern

### Factory Registration with Owner Tracking

```typescript
registerContextEngineForOwner(
  id: string,
  factory: ContextEngineFactory,
  owner: string,
  opts?: { allowSameOwnerRefresh?: boolean }
): ContextEngineRegistrationResult
```

### Resolution with Legacy Proxy

```typescript
resolveContextEngine(config?): ContextEngine
// 1. Check config.plugins.slots.contextEngine
// 2. Fall back to default slot ("legacy")
// 3. Wrap with sessionKey compat proxy
```

---

## 9. CLI Bootstrap Pattern

### Precomputed Help Fast-Path

```typescript
// openclaw.mjs
if (argv.includes("--help")) {
  // Load from dist/cli-startup-metadata.json instead of full bootstrap
}
```

### Fallback Chain

```typescript
tryImport(dist/entry.js)
  -> tryImport(dist/entry.mjs)
  -> tryImport(src/entry.ts)  // Source fallback with error guidance
```

### Module Compile Cache

```typescript
module.enableCompileCache()
```

---

## 10. Skill Platform Pattern (Progressive Disclosure)

### Markdown-First Skill Definition

```yaml
---
name: skill-name
description: "When to use this skill + specific triggers"
metadata:
  openclaw:
    requires:
      bins: ["gh"]
      env: ["API_KEY"]
    install:
      - id: brew
        kind: brew
        formula: gh
---
# SKILL.md body - loaded after skill triggers
```

### Tiered Loading

1. **Metadata** (~100 words) - Always in context for triggering
2. **SKILL.md body** (<5k words) - Loaded after skill triggers
3. **Bundled resources** - Loaded as needed by agent

---

## 11. Config Health Monitoring

### Fingerprinting

```typescript
type ConfigHealthFingerprint = {
  hash: string;
  bytes: number;
  mtimeMs: number | null;
  ctimeMs: number | null;
  hasMeta: boolean;
  gatewayMode: string | null;
  observedAt: string;
};
```

### Anomaly Detection

- Config size drop detection (flags when config shrinks >50%)
- Missing meta detection
- Gateway mode removal detection
- Automatic clobbered config snapshots

---

## 12. Error Handling Patterns

### Typed Error Codes with Cause Chain

```typescript
export class AcpRuntimeError extends Error {
  readonly code: AcpRuntimeErrorCode;
  override readonly cause?: unknown;
}
```

### Error Boundary Helpers

```typescript
withAcpRuntimeErrorBoundary()  // Clean try/catch wrapper
isAcpRuntimeError(value)      // Type guard
toAcpRuntimeError()            // Conversion utility
```

---

## 13. Temp File Lifecycle

```typescript
MEDIA_MAX_BYTES = 5 * 1024 * 1024  // 5MB default
MEDIA_FILE_MODE = 0o644            // Docker-readable
DEFAULT_TTL_MS = 2 * 60 * 1000    // 2 minutes temp file TTL
```

**Pattern:** Short TTL + world-readable permissions for Docker sandbox access.

---

## 14. Streaming Response Pattern (OpenAI WebSocket)

```typescript
// Per-session registry keyed by sessionId
const openaiWsManager = new OpenAIWebSocketManager(sessionId);

// Tracks previous_response_id for incremental updates
// Transparent fallback to HTTP on WS failure
```

---

## 15. WebSocket Log Sanitization

```typescript
replaceControlChars()  // 0x00-0x1F, 0x7F-0x9F -> spaces
sanitizeLogValue()     // Format chars removed, whitespace normalized, 300 char limit
```

---

## 16. Capability Advertisement (Device Nodes)

```typescript
InvokeCommandRegistry.advertisedCapabilities:
// - Canvas (always)
// - Device (always)
// - Notifications (always)
// - Camera (CameraEnabled)
// - SMS (SmsAvailable)
// - VoiceWake (VoiceWakeEnabled)
```

**Pattern:** Runtime flags determine which commands are available at runtime, not compile-time.

---

## Pattern Summary Table

| Pattern | Purpose | Location |
|---------|---------|----------|
| Plugin entry factory | Unified plugin lifecycle | `src/plugin-sdk/core.ts` |
| Channel composition | Shared behavior, platform-specific impl | `createChatChannelPlugin()` |
| ACP RPC over WS | Request/response with idempotency | `src/acp/`, `src/gateway/` |
| Block streaming coalescing | Reduce chat spam | `block-reply-pipeline.ts` |
| Session lanes + sandbox | Isolation for untrusted sessions | `agents/lanes.js`, `sandbox/` |
| Lazy promise caching | Fast startup, deferred deps | Throughout |
| External content wrapping | Injection prevention | `external-content.ts` |
| Context engine registry | Pluggable AI backends | `context-engine/registry.ts` |
| CLI bootstrap fast-path | Fast --help | `openclaw.mjs` |
| Progressive disclosure skills | Token-efficient knowledge | `skills/*/SKILL.md` |
| Config health fingerprint | Detect corruption | `config/io.ts` |
| Typed error codes | Structured errors with cause | Throughout |
| TTL temp files + 0o644 | Docker-compatible cleanup | `media/store.ts` |
| Capability advertisement | Runtime feature discovery | `InvokeCommandRegistry` |
