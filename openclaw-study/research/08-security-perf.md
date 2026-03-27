# OpenClaw Security and Performance Analysis

## Executive Summary

OpenClaw implements a comprehensive security model centered on a **personal assistant / trusted operator paradigm** rather than a multi-tenant model. The gateway binds to loopback by default (`127.0.0.1:18789`) and treats all authenticated callers as trusted operators. Security boundaries are enforced through authentication, exec approvals, sandboxing, and filesystem path guards rather than process-level isolation.

## 1. Security Model

### 1.1 Trust Paradigm

OpenClaw operates on a **one-user trusted-operator model** ([SECURITY.md:91-106](SECURITY.md#91)):

- Authenticated Gateway callers are treated as **trusted operators** for that gateway instance
- Session identifiers (`sessionKey`) are routing controls, not per-user authorization boundaries
- If one operator can view data from another operator on the same gateway, that is **expected behavior**
- Recommended mode: one user per machine/host, one gateway for that user
- For multiple users: use separate VPS or OS user/host boundaries

### 1.2 Gateway Authentication

The gateway supports multiple authentication modes ([src/gateway/auth.ts](src/gateway/auth.ts)):

| Mode | Description |
|------|-------------|
| `token` | Bearer token authentication (recommended) |
| `password` | Password-based authentication |
| `trusted-proxy` | Delegated auth via reverse proxy (Pomerium, Caddy, nginx) |
| `none` | No authentication (only for loopback with explicit configuration) |

**Auth Resolution Precedence**: `gateway.auth.mode` override -> config value -> password -> token -> default token

**Timing-Safe Comparison**: Password/token comparison uses `safeEqualSecret()` with SHA-256 hashing to prevent timing attacks ([src/security/secret-equal.ts:3-12](src/security/secret-equal.ts#L3-L12)).

### 1.3 Rate Limiting

Auth rate limiting protects against brute-force attacks ([src/gateway/auth-rate-limit.ts:1-17](src/gateway/auth-rate-limit.ts#L1-L17)):

- **Algorithm**: Sliding window in-memory rate limiter
- **Default limits**: 10 attempts per minute, 5-minute lockout
- **Loopback exemption**: Localhost addresses are exempt by default to prevent CLI lockout
- **Scope support**: Separate counters for shared-secret vs device-token auth
- **Auto-pruning**: Stale entries cleaned up every 60 seconds

### 1.4 Device Pairing and Identity

WebSocket connections require device identity and pairing ([docs/concepts/architecture.md:91-104](docs/concepts/architecture.md#L91-L104)):

- All WS clients include a **device identity** on `connect`
- New device IDs require pairing approval
- Gateway issues a **device token** for subsequent connects
- Local connects (loopback/tailnet) can be **auto-approved** for UX
- All connects must sign the `connect.challenge` nonce (v3 signature binds platform + deviceFamily)
- Metadata changes on reconnect require **repair pairing**

### 1.5 Plugin Trust Model

Plugins are treated as **part of the trusted computing base** ([SECURITY.md:108-114](SECURITY.md#L108-L114)):

- Plugins are loaded **in-process** with the Gateway
- Plugin behavior (reading env/files, running host commands) is **expected** inside this trust boundary
- Security reports must show a **boundary bypass** (unauthenticated plugin load, allowlist bypass, sandbox bypass)
- Installing/enabling a plugin grants it the **same trust level as local code**
- Use `plugins.allow` to pin explicit trusted plugin IDs

## 2. Secrets Management

### 2.1 detect-secrets Integration

Automated secret detection in CI/CD ([.detect-secrets.cfg](.detect-secrets.cfg)):

- Uses `detect-secrets==1.5.0` for baseline scanning
- Baseline file: `.secrets.baseline`
- Extensive exclusion patterns for false positives (Docker signatures, Sparkle appcast, test fixtures)
- Scan command: `detect-secrets scan --baseline .secrets.baseline`

### 2.2 Secrets Runtime

Secrets are resolved at runtime through a structured snapshot system ([src/secrets/runtime.ts](src/secrets/runtime.ts)):

- `prepareSecretsRuntimeSnapshot()` creates isolated config snapshots with resolved secrets
- `activateSecretsRuntimeSnapshot()` sets the active runtime config
- Supports auth profile stores per agent directory
- Refresh handler for hot-reloading secrets without restart

### 2.3 Secret Input Handling

Plugin SDK provides typed secret input schemas ([src/plugin-sdk/secret-input.ts](src/plugin-sdk/secret-input.ts)):

- `SecretInput<T>` type for secure credential handling
- Schema-based validation for secret values
- Support for environment variable references

## 3. Input Validation and Sanitization

### 3.1 Prompt Injection Prevention

OpenClaw explicitly acknowledges that **the model/agent is not a trusted principal** ([SECURITY.md:159-165](SECURITY.md#L159-L165)):

> Assume prompt/content injection can manipulate behavior. Security boundaries come from host/config trust, auth, tool policy, sandboxing, and exec approvals.

**Sanitization functions** ([src/agents/sanitize-for-prompt.ts](src/agents/sanitize-for-prompt.ts)):

- `sanitizeForPromptLiteral()`: Strips Unicode control (Cc) + format (Cf) characters
- Removes CR/LF/NUL, bidirectional marks, zero-width characters
- `wrapUntrustedPromptDataBlock()`: Wraps untrusted content in data blocks with HTML escaping

### 3.2 Channel Plugin Input Validation

Channel plugins implement `validateInput` in their setup adapters ([src/channels/plugins/types.adapters.ts:84-88](src/channels/plugins/types.adapters.ts#L84-L88)):

```typescript
validateInput?: (params: {
  cfg: OpenClawConfig;
  accountId: string;
  input: ChannelSetupInput;
}) => string | null;
```

### 3.3 WebSocket Message Sanitization

WS connection handling includes log value sanitization ([src/gateway/server/ws-connection.ts:29-59](src/gateway/server/ws-connection.ts#L29-L59)):

- `replaceControlChars()`: Replaces control characters (0x00-0x1F, 0x7F-0x9F) with spaces
- `sanitizeLogValue()`: Removes format control characters, normalizes whitespace, truncates to 300 chars

## 4. SQL Injection Prevention

### 4.1 SQLite with Prepared Statements

OpenClaw uses Node.js built-in `node:sqlite` with prepared statements ([src/memory/sqlite.ts](src/memory/sqlite.ts)):

- Uses `node:sqlite` (built-in, no external dependency)
- Vector search extension via `sqlite-vec` ([src/memory/sqlite-vec.ts](src/memory/sqlite-vec.ts))
- Parameterized queries for all SQL operations

### 4.2 Session Key Validation

Session keys have strict format validation ([src/routing/session-key.ts:24-27](src/routing/session-key.ts#L24-L27)):

```typescript
const VALID_ID_RE = /^[a-z0-9][a-z0-9_-]{0,63}$/i;
```

- Alphanumeric with hyphens/underscores, 0-63 chars after initial char
- Invalid characters collapsed to "-" during normalization
- Prevents path traversal via session key manipulation

## 5. WebSocket Security

### 5.1 Default Binding

Gateway WebSocket binds to **loopback only** by default ([docs/concepts/architecture.md:15-16](docs/concepts/architecture.md#L15-L16)):

```
ws://127.0.0.1:18789
```

### 5.2 Origin and Host Header Validation

WS connection handler validates request origins ([src/gateway/server/ws-connection.ts:115-128](src/gateway/server/ws-connection.ts#L115-L128)):

- Tracks `hostHeaderFallbackAccepted` metric for security auditing
- Validates `Origin` and `Host` headers
- Proxy-aware IP resolution for `X-Forwarded-For`

### 5.3 Handshake Timeout

Pre-auth handshake has a configurable timeout ([src/gateway/server/ws-connection.ts:268-278](src/gateway/server/ws-connection.ts#L268-L278)):

```typescript
const handshakeTimeoutMs = getPreauthHandshakeTimeoutMsFromEnv();
```

- Connections that fail to complete handshake within timeout are closed
- Default: 30 seconds (configurable via environment)

### 5.4 Nonce Challenge

All WS connections must sign a challenge nonce to prevent replay attacks ([docs/concepts/architecture.md:98-101](docs/concepts/architecture.md#L98-L101)):

- Gateway sends `connect.challenge` with nonce after socket open
- Client must sign and return the nonce
- Signature payload v3 binds platform + deviceFamily

## 6. Plugin Sandboxing

### 6.1 Sandbox Modes

Exec behavior has configurable sandbox modes ([SECURITY.md:103-106](SECURITY.md#L103-L106)):

| Mode | Description |
|------|-------------|
| `off` (default) | Exec runs directly on gateway host |
| `non-main` | Only non-main agents run in sandbox |
| `all` | All agents run in sandbox |

### 6.2 Tool Execution Sandboxing

Plugin commands run via `runPluginCommandWithTimeout()` ([src/plugin-sdk/run-command.ts](src/plugin-sdk/run-command.ts)):

- Timeout enforcement for plugin commands
- Normalized stdout/stderr capture
- Exit code propagation

### 6.3 Filesystem Bridge

Sandbox filesystem access is controlled via `SandboxFsBridge` ([src/agents/sandbox/fs-bridge.ts](src/agents/sandbox/fs-bridge.ts)):

- Path safety validation for sandbox operations
- `workspaceOnly` mode restricts writes to configured workspace
- Anchored operations for dangerous mutations

### 6.4 Temp Directory Isolation

Dedicated temp root with security checks ([src/infra/tmp-openclaw-dir.ts](src/infra/tmp-openclaw-dir.ts)):

- Preferred: `/tmp/openclaw` (with 0o700 permissions)
- Fallback: `os.tmpdir()/openclaw-<uid>` on multi-user hosts
- Permission verification: checks owner UID and group/other writability
- Auto-repair: attempts to chmod if world/group writable
- Rejects fallback dir if unsafe and cannot be repaired

## 7. SSRF Protection

Server-side request forgery protection ([src/infra/net/ssrf.ts](src/infra/net/ssrf.ts)):

### 7.1 Blocked Hostnames

```typescript
const BLOCKED_HOSTNAMES = new Set([
  "localhost",
  "localhost.localdomain",
  "metadata.google.internal",
]);
```

### 7.2 Policy Options

```typescript
type SsrFPolicy = {
  allowPrivateNetwork?: boolean;
  dangerouslyAllowPrivateNetwork?: boolean;
  allowRfc2544BenchmarkRange?: boolean;
  allowedHostnames?: string[];
  hostnameAllowlist?: string[];
};
```

### 7.3 Private Network Blocking

- RFC1918 addresses (10.x, 172.16-31.x, 192.168.x)
- Link-local, ULA IPv6, CGNAT (100.64/10)
- Special-use addresses (loopback, multicast, broadcast)

## 8. Exec Approvals

### 8.1 Approval Security Levels

```typescript
export type ExecSecurity = "deny" | "allowlist" | "full";
```

- `deny`: Default, blocks exec
- `allowlist`: Only approved commands can execute
- `full`: No exec restrictions

### 8.2 Approval Binding

Exec approvals bind to concrete context ([SECURITY.md:172-175](SECURITY.md#L172-L175)):

- Exact command/cwd/env context
- Best-effort file operand snapshot (SHA-256 when identifiable)
- NOT a complete semantic model of interpreter load paths

### 8.3 Safe Bin Profiles

Executable allowlisting with hardened profiles for interpreters ([src/infra/exec-safe-bin-runtime-policy.ts](src/infra/exec-safe-bin-runtime-policy.ts)):

- `strictInlineEval`: Prevents eval-like constructs in interpreters
- Safe bin trusted directories validation
- Risky directory detection (/tmp, /var/tmp, home bin directories)

## 9. Memory Security

### 9.1 Memory Index Manager

Per-agent memory managers with isolated caches ([src/memory/manager.ts:39-54](src/memory/manager.ts#L39-L54)):

- Global singleton cache keyed by agent ID
- Cleanup on module reset via `closeAllMemoryIndexManagers()`

### 9.2 Memory as Trusted State

`MEMORY.md` and `memory/*.md` are treated as **trusted local operator state** ([SECURITY.md:178-185](SECURITY.md#L178-L185)):

- Editing memory files = crossing trusted operator boundary
- Memory search over those files is expected behavior
- Not a security boundary

## 10. Performance Patterns

### 10.1 Streaming Responses

OpenAI WebSocket streaming integration ([src/agents/openai-ws-stream.ts](src/agents/openai-ws-stream.ts)):

- Per-session `OpenAIWebSocketManager` keyed by sessionId
- Tracks `previous_response_id` for incremental updates
- Transparent fallback to HTTP `streamSimple` on WS failure
- Per-session registry with cleanup on dispose

### 10.2 Caching Strategies

**Embedding Cache**: In-memory cache for vector embeddings ([src/memory/manager.ts:36](src/memory/manager.ts#L36)):

```typescript
const EMBEDDING_CACHE_TABLE = "embedding_cache";
```

**Global Singletons**: `resolveGlobalSingleton<T>()` pattern for process-wide caches

**Session Registry**: Module-level Map registries (e.g., `wsRegistry` for WebSocket sessions)

### 10.3 Batch Operations

Memory indexing supports batch operations ([src/memory/manager.ts:95-100](src/memory/manager.ts#L95-L100)):

```typescript
private batch: {
  enabled: boolean;
  wait: boolean;
  concurrency: number;
  pollIntervalMs: number;
  timeoutMs: number;
};
```

### 10.4 Lazy Loading

Dynamic imports used for lazy module loading:

```typescript
// Example pattern from src/security/audit.ts
channelPluginsModulePromise ??= import("../channels/plugins/index.js");
```

Module-level promise caching prevents duplicate dynamic imports.

### 10.5 Idempotency

Gateway protocol requires idempotency keys for side-effecting methods ([docs/concepts/architecture.md:87-88](docs/concepts/architecture.md#L87-L88)):

> Idempotency keys are required for side-effecting methods (`send`, `agent`) to safely retry; the server keeps a short-lived dedupe cache.

## 11. Configuration Security Auditing

### 11.1 Security Audit Command

`openclaw security audit` performs comprehensive checks ([src/security/audit.ts](src/security/audit.ts)):

- Filesystem permissions (state dir, config file)
- Gateway bind + auth configuration
- Browser control auth
- Exec runtime settings
- Channel security settings
- Hooks hardening
- Dangerous config flags
- Plugin trust findings
- Deep audit probes live gateway

### 11.2 Key Audit Findings

| Check ID | Severity | Description |
|----------|----------|-------------|
| `gateway.bind_no_auth` | critical | Gateway binds beyond loopback without auth |
| `gateway.control_ui.device_auth_disabled` | critical | Device auth disabled for Control UI |
| `gateway.tailscale_funnel` | critical | Tailscale Funnel exposes gateway publicly |
| `gateway.token_too_short` | warn | Gateway token < 24 chars |
| `tools.exec.security_full_configured` | warn/critical | Full exec trust enabled |
| `fs.state_dir.perms_world_writable` | critical | State dir world-writable |

## 12. Docker Security

Running OpenClaw in Docker ([SECURITY.md:266-280](SECURITY.md#L266-L280)):

- Official image runs as non-root user (`node`)
- `--read-only` flag recommended for filesystem protection
- `--cap-drop=ALL` to limit container capabilities

```bash
docker run --read-only --cap-drop=ALL \
  -v openclaw-data:/app/data \
  openclaw/openclaw:latest
```

## 13. Security Documentation

### 13.1 Trust Repository

Separate trust/threat model repository: [openclaw/trust](https://github.com/openclaw/trust)

### 13.2 Security Contact

- Core CLI/Gateway: [openclaw/openclaw](https://github.com/openclaw/openclaw)
- Report to: [security@openclaw.ai](mailto:security@openclaw.ai)
- Full reporting instructions: [trust.openclaw.ai](https://trust.openclaw.ai)

### 13.3 Required in Security Reports

1. Title
2. Severity Assessment
3. Impact
4. Affected Component
5. Technical Reproduction
6. Demonstrated Impact
7. Environment
8. Remediation Advice

## 14. Key Security Recommendations

Based on the security model, operators should:

1. **Keep gateway on loopback** unless accessing via Tailscale VPN
2. **Use strong gateway tokens** (>24 characters)
3. **Enable sandbox mode** (`agents.defaults.sandbox.mode: "non-main"`) for agents handling untrusted input
4. **Configure exec allowlist** rather than `exec.security: "full"`
5. **Pin explicit plugin IDs** via `plugins.allow`
6. **Use Tailscale Serve** (`tailscale.mode: "serve"`) rather than Funnel for remote access
7. **Set `tools.exec.applyPatch.workspaceOnly: true`** to restrict filesystem writes
8. **Run as single user** per gateway instance
9. **Keep browser control behind gateway auth** - no separate browser-only auth
10. **Monitor `openclaw security audit --deep`** for configuration warnings

## Summary

OpenClaw's security architecture reflects its personal assistant use case: strong local isolation defaults, comprehensive auth with rate limiting, plugin trust boundaries, and exec approval workflows. The system is not designed for multi-tenant adversarial environments but provides solid security for single-user trusted-operator deployments with defense-in-depth features for hardening when needed.