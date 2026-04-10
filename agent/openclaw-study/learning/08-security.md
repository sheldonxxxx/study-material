# OpenClaw Security Model

## Trust Paradigm: One-User Trusted Operator

OpenClaw operates on a **one-user trusted-operator model**. Authenticated Gateway callers are treated as trusted operators for that Gateway instance. This is a personal assistant paradigm, not a multi-tenant environment.

**Key implications:**
- Session identifiers (`sessionKey`) are routing controls, not authorization boundaries
- If one operator can view data from another operator on the same Gateway, that is expected behavior
- Recommended: one user per machine/host, one Gateway per user
- For multiple users: use separate VPS or OS user/host boundaries

## Gateway Authentication

| Mode | Description |
|------|-------------|
| `token` | Bearer token authentication (recommended) |
| `password` | Password-based authentication |
| `trusted-proxy` | Delegated auth via reverse proxy (Pomerium, Caddy, nginx) |
| `none` | No authentication (only for loopback with explicit configuration) |

**Auth Resolution Precedence**: `gateway.auth.mode` override -> config value -> password -> token -> default token

**Security features:**
- **Timing-safe comparison**: Uses `safeEqualSecret()` with SHA-256 hashing to prevent timing attacks
- **Rate limiting**: Sliding window in-memory limiter, 10 attempts/minute, 5-minute lockout
- **Loopback exemption**: Localhost addresses exempt by default to prevent CLI lockout

## Device Pairing and Identity

WebSocket connections require device identity and pairing:

1. All WS clients include a **device identity** on `connect`
2. New device IDs require pairing approval
3. Gateway issues a **device token** for subsequent connections
4. Local connections (loopback/tailnet) can be **auto-approved** for UX
5. All connections must sign the `connect.challenge` nonce (v3 signature binds platform + deviceFamily)
6. Metadata changes on reconnect require **repair pairing**

## Plugin Trust Model

Plugins are treated as **part of the trusted computing base**:

- Plugins are loaded **in-process** with the Gateway
- Plugin behavior (reading env/files, running host commands) is **expected** inside this trust boundary
- Installing/enabling a plugin grants it the **same trust level as local code**
- Use `plugins.allow` to pin explicit trusted plugin IDs

**Security boundary bypass** would require: unauthenticated plugin load, allowlist bypass, or sandbox bypass.

## Prompt Injection Prevention

OpenClaw explicitly acknowledges that **the model/agent is not a trusted principal**. Security boundaries come from host/config trust, auth, tool policy, sandboxing, and exec approvals.

**Sanitization functions:**
- `sanitizeForPromptLiteral()`: Strips Unicode control (Cc) + format (Cf) characters, removes CR/LF/NUL, bidirectional marks, zero-width characters
- `wrapUntrustedPromptDataBlock()`: Wraps untrusted content in data blocks with HTML escaping

## WebSocket Security

**Default binding**: Gateway WebSocket binds to loopback only (`ws://127.0.0.1:18789`)

**Security measures:**
- Origin and Host header validation
- Handshake timeout (30 seconds default, configurable)
- Nonce challenge requirement for all connections
- Log value sanitization (control character replacement, 300-char truncation)

## SSRF Protection

Blocked hostnames include: `localhost`, `localhost.localdomain`, `metadata.google.internal`

**Policy options:**
```typescript
type SsrFPolicy = {
  allowPrivateNetwork?: boolean;
  dangerouslyAllowPrivateNetwork?: boolean;
  allowRfc2544BenchmarkRange?: boolean;
  allowedHostnames?: string[];
  hostnameAllowlist?: string[];
};
```

**Private network blocking:**
- RFC1918 addresses (10.x, 172.16-31.x, 192.168.x)
- Link-local, ULA IPv6, CGNAT (100.64/10)
- Special-use addresses (loopback, multicast, broadcast)

## Exec Approvals

**Security levels:**
```typescript
export type ExecSecurity = "deny" | "allowlist" | "full";
```

- `deny`: Default, blocks exec
- `allowlist`: Only approved commands can execute
- `full`: No exec restrictions

**Safe bin profiles** with `strictInlineEval` prevent eval-like constructs in interpreters.

## Sandbox Modes

| Mode | Description |
|------|-------------|
| `off` (default) | Exec runs directly on gateway host |
| `non-main` | Only non-main agents run in sandbox |
| `all` | All agents run in sandbox |

**Filesystem bridge** (`SandboxFsBridge`) controls access:
- Path safety validation for sandbox operations
- `workspaceOnly` mode restricts writes to configured workspace

**Temp directory isolation:**
- Preferred: `/tmp/openclaw` (0o700 permissions)
- Fallback: `os.tmpdir()/openclaw-<uid>` on multi-user hosts
- Permission verification and auto-repair

## Secrets Management

**detect-secrets integration:**
- Uses `detect-secrets==1.5.0` for baseline scanning
- Extensive exclusion patterns for false positives (Docker signatures, Sparkle appcast, test fixtures)

**Runtime secrets:**
- `prepareSecretsRuntimeSnapshot()` creates isolated config snapshots with resolved secrets
- `activateSecretsRuntimeSnapshot()` sets the active runtime config
- Supports auth profile stores per agent directory

## SQL Injection Prevention

- Uses Node.js built-in `node:sqlite` with prepared statements
- Parameterized queries for all SQL operations
- Session key validation with strict format: `/^[a-z0-9][a-z0-9_-]{0,63}$/i`

## Docker Security

- Official image runs as non-root user (`node`)
- `--read-only` flag recommended for filesystem protection
- `--cap-drop=ALL` to limit container capabilities

```bash
docker run --read-only --cap-drop=ALL \
  -v openclaw-data:/app/data \
  openclaw/openclaw:latest
```

## Security Audit Command

`openclaw security audit` performs comprehensive checks:

| Check ID | Severity | Description |
|----------|----------|-------------|
| `gateway.bind_no_auth` | critical | Gateway binds beyond loopback without auth |
| `gateway.control_ui.device_auth_disabled` | critical | Device auth disabled for Control UI |
| `gateway.tailscale_funnel` | critical | Tailscale Funnel exposes gateway publicly |
| `gateway.token_too_short` | warn | Gateway token < 24 chars |
| `tools.exec.security_full_configured` | warn/critical | Full exec trust enabled |
| `fs.state_dir.perms_world_writable` | critical | State dir world-writable |

## Key Security Recommendations

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
