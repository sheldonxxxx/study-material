# My Action Items: Recommendations for a Similar Project

**Study Root:** `/Users/sheldon/Documents/claw/openclaw-study`
**Date:** 2026-03-26
**Source:** Synthesis from all research documents

---

## Key Decisions to Make Early

### 1. Trust Model

**Decision:** Define upfront whether your system is single-user or multi-user.

OpenClaw chose **single-user trusted-operator**. This simplifies:
- Session key design (no per-user authorization)
- Plugin trust (in-process, same trust as local code)
- Config management (single `~/.openclaw/`)

**If multi-user:** Add application-level user separation, per-user session isolation, and plugin sandboxing.

### 2. Plugin Architecture: In-Process vs Isolated

**Decision:** Will plugins run in-process or in separate processes?

OpenClaw chose **in-process** with high trust. This gives:
- Fast communication (direct function calls)
- Shared state easily
- Simple deployment

But requires:
- Plugin SDK stability (breaking changes affect plugins)
- Trust boundary at plugin installation

**Alternative:** MCP-style stdio communication for process isolation (like OpenClaw's `mcporter`).

### 3. Channel Normalization Strategy

**Decision:** Codify shared patterns vs per-channel autonomy.

OpenClaw's `createChatChannelPlugin` composes:
- Security (DM policy, allowFrom)
- Pairing (text-based pairing)
- Threading (reply-to modes)
- Outbound (delivery modes, chunking)

This consistency aids debugging but requires upfront design investment.

### 4. Local-First vs Cloud Dependency

**Decision:** What requires network? What is strictly local?

OpenClaw's local-first means:
- Gateway on loopback by default
- Sessions stored in `~/.openclaw/`
- No cloud dependency for core

**Trade-off:** Users self-manage; no automatic backup/sync; harder to access remotely without Tailscale.

---

## Patterns to Adopt

### 1. Plugin SDK with Dual Registration

```typescript
definePluginEntry({
  register(api) {
    // Setup-only registration
    if (api.registrationMode !== "full") return;
    // Full runtime registration
  }
});
```

**Benefit:** Setup wizards render without activating runtime.

### 2. Block Streaming with Coalescing

```typescript
createBlockReplyPipeline({
  coalescing: {
    maxChars: 200,
    idleMs: 1000,
    flushOnParagraph: true
  }
});
```

**Benefit:** Reduces message spam; respects platform limits.

### 3. Lazy Loading with Module-Level Promise Cache

```typescript
let modulePromise: Promise<Module> | undefined;
export function getModule() {
  modulePromise ??= import("./module");
  return modulePromise;
}
```

**Benefit:** Fast startup; deferred heavy dependency loading.

### 4. Typed Error Codes with Cause Chain

```typescript
class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    options?: { cause?: unknown }
  ) {
    super(message, options);
  }
}
```

**Benefit:** Structured errors; stack trace preservation; easy error categorization.

### 5. Config Health Monitoring

```typescript
type ConfigHealthFingerprint = {
  hash: string;
  bytes: number;
  mtimeMs: number;
};
// Detect size drops, missing meta, corruption
```

**Benefit:** Early detection of config corruption or accidental changes.

### 6. External Content Wrapping

```typescript
wrapUntrustedContent(content, {
  markerId: randomBytes(8).toString("hex"),
  suspiciousPatterns: ["ignore previous", "new instructions:"]
});
```

**Benefit:** Prompt injection detection and isolation.

### 7. Capability Advertisement Pattern

```typescript
advertisedCapabilities: {
  canvas: "always",
  camera: runtimeFlags.CameraEnabled ? "CameraEnabled" : false,
  // ...
}
```

**Benefit:** Runtime feature discovery without compile-time constants.

### 8. CLI Bootstrap with Fallback Chain

```typescript
tryImport(dist/entry.js)
  .catch(() => tryImport(dist/entry.mjs))
  .catch(() => tryImportWithHelp(src/entry.ts));
```

**Benefit:** Graceful degradation; helpful error messages.

### 9. Session Lane Isolation

```typescript
const AGENT_LANE_MAIN = "main";
const AGENT_LANE_SUBAGENT = "subagent";
// Main = elevated privileges, Subagent = sandboxed
```

**Benefit:** Clear separation of trusted vs untrusted execution contexts.

### 10. Secrets Detection Baseline

- Create `.secrets.baseline` early
- Run detect-secrets in pre-commit
- Document false positive exclusions

---

## Things to Avoid

### 1. Monolithic Core Files

**Avoid:** Files over 500 lines that handle multiple concerns.

**Instead:** Break into focused modules with clear interfaces.

### 2. Eager Heavy Dependency Loading

**Avoid:** `import Discord from "discord.js"` at module load time.

**Instead:** `await import("discord.js")` with promise caching when first needed.

### 3. Implicit Session State Leaking

**Avoid:** Auth profile overrides persisting across sessions.

**Instead:** Explicit session cleanup; consider immutable session state.

### 4. Inconsistent Channel Patterns

**Avoid:** Each channel evolving its own conventions.

**Instead:** Codify shared patterns in `createChatChannelPlugin()` early.

### 5. No Test Coverage for Extensions

**Avoid:** Treating extensions as "not our code."

**Instead:** At minimum, test the plugin SDK contract.

### 6. Undocumented Trust Boundaries

**Avoid:** Implicit trust assumptions.

**Instead:** Write SECURITY.md defining:
- What is trusted
- What can插件 access
- How to report issues

### 7. Web-Based Messaging Without Fallback

**Avoid:** WhatsApp-style QR code login for production features.

**Instead:** Use official APIs; plan for platform changes.

---

## Architecture Choices to Consider

### 1. Gateway: Single Process vs Distributed

OpenClaw: **Single process gateway** on localhost.

**Consider:**
- Single process: Simpler, lower latency, shared memory
- Distributed: Better isolation, scales across machines, operational complexity

### 2. Session Storage: In-Memory vs Persistence

OpenClaw: **Persistent JSON files** in `~/.openclaw/sessions/`.

**Consider:**
- Files: Simple, inspectable, no DB setup
- SQLite: Better query capability, concurrent access
- Memory + periodic write: Lower latency, requires durability strategy

### 3. WebSocket vs HTTP for Gateway Communication

OpenClaw: **WebSocket primary** with HTTP for webhooks.

**Consider:**
- WebSocket: Bidirectional, lower overhead per message, connection state
- HTTP: Simpler, better observability, stateless
- Both: WebSocket for interactive, HTTP for webhooks/triggers

### 4. AI Integration: Provider Abstraction vs Direct

OpenClaw: **Context engine registry** with pluggable providers.

**Benefit:** Users can swap providers; fallback chains.

**Cost:** Abstraction overhead; not all features available on all providers.

### 5. Sandbox: Docker vs Process Isolation vs No Sandbox

OpenClaw: **Docker for non-main sessions**, host for main.

**Consider:**
- Docker: Good isolation, heavy dependency
- Native sandbox (Node VM): Lighter, less complete isolation
- No sandbox: Maximum performance, minimum safety

---

## Tooling Choices

### 1. TypeScript with Strict Mode

```json
{
  "compilerOptions": {
    "strict": true,
    "noEmitOnError": true
  }
}
```

**Rationale:** Catches bugs early; self-documenting code.

### 2. Vitest over Jest

**Rationale:** Faster, native ESM, V8 coverage, fork-based parallelism.

### 3. Oxlint over ESLint

**Rationale:** Faster, fewer plugins needed, good TypeScript support.

### 4. pnpm Workspaces

**Rationale:** Fast installs, strict dependency hoisting, good monorepo support.

### 5. tsdown for Bundling

**Rationale:** Simple, fast, good TypeScript support.

### 6. detect-secrets for Secret Scanning

**Rationale:** Pre-commit + CI integration, baseline for allowlisting false positives.

---

## Practical Next Steps

### Phase 1: Foundation (Week 1-2)

1. **Initialize monorepo structure**
   - pnpm workspaces
   - TypeScript strict mode
   - Vitest setup with coverage thresholds
   - Pre-commit hooks (detect-secrets, lint, format)

2. **Design plugin SDK**
   - Define plugin entry interface
   - Implement dual registration (setup/full)
   - Write SDK documentation

3. **Build minimal gateway**
   - WebSocket server on loopback
   - Session management with lanes
   - Config with health monitoring

### Phase 2: Core Channels (Week 3-4)

4. **Implement 2-3 channel plugins**
   - Use `createChatChannelPlugin()` composition
   - Document normalization patterns
   - Add tests for SDK contract

5. **Implement block streaming**
   - Coalescing pipeline
   - Per-channel chunking limits
   - Platform-specific delivery

### Phase 3: AI Integration (Week 5-6)

6. **Build context engine registry**
   - Factory registration pattern
   - Legacy proxy for compatibility
   - At least 2 provider implementations

7. **Implement session isolation**
   - Lane system
   - Sandbox runtime resolution
   - Tool allowlists

### Phase 4: Polish (Week 7-8)

8. **Security hardening**
   - External content wrapping
   - Secret scanning baseline
   - SECURITY.md documentation

9. **Documentation**
   - CLAUDE.md for AI assistants
   - CONTRIBUTING.md
   - PR template

10. **CLI bootstrap**
    - Precomputed help fast-path
    - Graceful fallback chain
    - Module compile cache

---

## Quick Reference: OpenClaw Patterns Summary

| Pattern | When to Use | Key Benefit |
|---------|-------------|-------------|
| Plugin SDK | Extensibility required | Clean separation |
| Dual registration | Setup vs runtime differ | Faster startup |
| Channel composition | Shared behavior across platforms | Consistency |
| Block coalescing | Streaming text to chat | Reduces spam |
| Session lanes | Multiple trust levels | Clear isolation |
| Lazy promise cache | Heavy dependencies | Fast startup |
| External wrapping | Untrusted input | Injection prevention |
| Config fingerprinting | User-managed config | Corruption detection |
| Capability advertisement | Runtime feature discovery | Flexibility |
| Typed error codes | Production systems | Debugging |
