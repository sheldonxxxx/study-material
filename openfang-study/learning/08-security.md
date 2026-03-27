# OpenFang Security Assessment

## Security Overview

OpenFang implements **defense-in-depth** with 16 independent security systems spanning capability-based access control, WASM sandboxing, cryptographic audit trails, taint tracking, and network protection. The project takes security seriously enough to have a dedicated security contact (jaber@rightnowai.co) and a detailed 1,493-line security architecture document.

**Security Score: 16 discrete systems** -- highest in the agent framework landscape.

## Most Impressive Security Measures

### 1. WASM Dual-Metered Sandbox

Tool code runs inside WebAssembly with **two independent metering mechanisms** running simultaneously:

- **Fuel Metering:** Counts WASM instructions deterministically. When exhausted, execution traps with `Trap::OutOfFuel`.
- **Epoch Interruption:** Watchdog thread sleeps for configured timeout, then increments engine epoch. When deadline passes, execution traps with `Trap::Interrupt`.

This dual approach is defense-in-depth at the VM level: fuel catches CPU-intensive loops while epochs catch host-call abuse or environmental slowdowns.

```rust
// Default limits
pub fuel_limit: u64 = 1_000_000;      // Instruction budget
pub max_memory_bytes: usize = 16 MB;  // Memory limit
pub timeout_secs: u64 = 30;           // Wall-clock timeout
```

### 2. Merkle Hash-Chain Audit Trail

Every security-critical action is appended to a tamper-evident chain where each entry contains the SHA-256 hash of its own contents concatenated with the previous entry's hash.

```rust
pub struct AuditEntry {
    pub seq: u64,
    pub timestamp: String,
    pub agent_id: String,
    pub action: AuditAction,
    pub detail: String,
    pub outcome: String,
    pub prev_hash: String,
    pub hash: String,
}
```

Tampering with any entry breaks the entire chain. Verification recomputes every hash and checks linkage. The audit log uses poison-safe mutex recovery (`unwrap_or_else`) to remain available after panics.

### 3. Information Flow Taint Tracking

Lattice-based taint propagation prevents tainted data from flowing into sensitive sinks:

| TaintLabel | Source |
|------------|--------|
| `ExternalNetwork` | External network requests |
| `UserInput` | Direct user input |
| `Pii` | Personally identifiable information |
| `Secret` | API keys, tokens, passwords |
| `UntrustedAgent` | Sandbox/untrusted agents |

Taint sinks block labeled data from reaching sensitive operations:
- `shell_exec()` blocks: `ExternalNetwork`, `UntrustedAgent`, `UserInput`
- `net_fetch()` blocks: `Secret`, `Pii`
- `agent_message()` blocks: `Secret`

### 4. SSRF Protection with DNS Rebinding Defense

The `host_net_fetch` function implements multiple layers:

1. **Scheme validation:** Only `http://` and `https://` allowed
2. **Hostname blocklist:** `localhost`, `metadata.google.internal`, `169.254.169.254`, etc.
3. **DNS resolution check:** Every resolved IP checked against private ranges
4. **Private IP detection:** Covers RFC 1918 (10.x, 172.16-31.x, 192.168.x), link-local, IPv6 unique local

The DNS resolution check defeats DNS rebinding attacks because the check happens after resolution, not on the hostname alone.

### 5. Secret Zeroization

Every API key field uses `Zeroizing<String>` from the `zeroize` crate. When dropped, memory is overwritten with zeros, preventing secrets from lingering in memory for memory forensics or core dump recovery.

Used in:
- All 3 LLM drivers (Anthropic, Gemini, OpenAI-compat)
- 8+ channel adapters (Discord, Email, Bluesky, DingTalk, Feishu, Flock, Gitter, Gotify)
- Web search modules
- Embedding clients

### 6. OFP Mutual Authentication

The OpenFang Wire Protocol uses HMAC-SHA256 with:
- **Mutual authentication:** Both sides prove knowledge of shared secret
- **Replay protection:** Random UUID nonces per handshake
- **Timing-attack resistance:** `subtle::ConstantTimeEq` for HMAC comparison
- **Mandatory secret:** OFP refuses to start without a configured `shared_secret`

### 7. Capability-Based Access Control

Agents can only perform actions explicitly granted in their manifest. Capabilities are immutable after agent creation and enforced at the kernel level before any tool invocation.

```rust
pub enum Capability {
    FileRead(String),       // Glob pattern
    FileWrite(String),
    NetConnect(String),     // Host:port pattern
    ToolInvoke(String),     // Specific tool ID
    LlmMaxTokens(u64),
    AgentSpawn,
    EconSpend(f64),
    // ... 20+ capability types
}
```

Child agents cannot exceed parent capabilities (privilege escalation prevention).

## Credential Management

### Environment Variables

API keys loaded from environment variables are **never** stored in plain strings -- all are wrapped in `Zeroizing<String>`:

```rust
fn resolve_api_key(env_var: &str) -> Option<Zeroizing<String>> {
    std::env::var(env_var).ok().filter(|k| !k.is_empty()).map(Zeroizing::new)
}
```

### AES-256-GCM Credential Vault

The `openfang-extensions` crate provides an AES-256-GCM credential vault for secure storage at rest.

### OAuth2 PKCE

OAuth2 with PKCE (Proof Key for Code Exchange) support for enterprise integrations.

### Subprocess Sandbox

When spawning child processes, `env_clear()` removes ALL inherited environment variables. Only an explicit allowlist of safe variables is passed:

**All platforms:**
```rust
pub const SAFE_ENV_VARS: &[&str] = &[
    "PATH", "HOME", "TMPDIR", "TMP", "TEMP", "LANG", "LC_ALL", "TERM",
];
```

This prevents API keys, database credentials, and other secrets from leaking to child processes.

## Authentication Patterns

### API Authentication

- Bearer token authentication for API access
- Loopback bypass when no API key configured (localhost-only mode)
- CORS restricted to localhost when no API key configured

### RBAC Multi-User

Owner/Admin/User/Viewer role hierarchy for multi-user deployments.

### Health Endpoint Redaction

| Endpoint | Auth Required | Data Exposed |
|----------|---------------|--------------|
| `GET /api/health` | No | `status`, `version` only |
| `GET /api/health/detail` | Yes | Full diagnostics, uptime, agent count, config warnings |

## Protection Against Common Attacks

| Attack | Protection |
|--------|------------|
| **SSRF** | Hostname blocklist + DNS resolution check + private IP blocking |
| **Path Traversal** | `safe_resolve_path()` / `safe_resolve_parent()` with canonicalization |
| **Shell Injection** | `Command::new` (no shell) + metacharacter handling |
| **Prompt Injection** | Taint tracking + prompt injection scanner for skills |
| **Data Exfiltration** | Taint sinks block Secret/PII to network sinks |
| **Privilege Escalation** | Capability inheritance validation |
| **Tampered Manifests** | Ed25519 signed manifests with SHA-256 content hash |
| **Tool Loops** | Loop guard with graduated response (warn/block/circuit-break) |
| **Session Corruption** | 7-phase session repair before LLM calls |
| **Memory Forensics** | `Zeroizing<String>` for all secret fields |
| **Timing Attacks** | `subtle::ConstantTimeEq` for HMAC verification |
| **XSS** | CSP, X-Frame-Options, X-Content-Type-Options headers |
| **Clickjacking** | X-Frame-Options: DENY |
| **MIME Sniffing** | X-Content-Type-Options: nosniff |
| **API Abuse** | GCRA rate limiting (500 tokens/min/IP) |
| **Unauthorized P2P** | HMAC-SHA256 mutual authentication |
| **Secret Leakage to Children** | `env_clear()` + allowlist |

## Security Headers (on all API responses)

```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' cdn.jsdelivr.net; ...
Referrer-Policy: strict-origin-when-cross-origin
Cache-Control: no-store, no-cache, must-revalidate
```

## Rate Limiting

GCRA (Generic Cell Rate Algorithm) via the `governor` crate:
- **500 tokens per minute per IP**
- Operation costs range from 1 (health check) to 100 (workflow runs)
- Stale entry cleanup via `DashMapStateStore`

## Biggest Security Concerns

### 1. Pre-1.0 Stability

> "Breaking changes may occur between minor versions until v1.0."

Security properties may change between versions without notice. Production deployments should pin to a specific commit.

### 2. Prompt Injection via Skills

The prompt injection scanner (`SkillVerifier::scan_prompt_content()`) detects common attack patterns in skill manifests and SKILL.md content, but:

- Detection is pattern-based (regex matching for override attempts, exfiltration patterns, shell references)
- Sophisticated attackers may craft injection that evades detection
- The scanner is a defense-in-depth layer, not a complete solution

### 3. WASM Sandbox Trust Model

WASM sandboxing limits tool code to declared capabilities, but:

- WASM bytecode is still executed -- sandbox escapes are theoretically possible
- The project acknowledges WASM sandbox escapes as in-scope for security reports
- 16MB memory limit and fuel/timeout limits constrain but do not eliminate risk

### 4. Secret Zeroization Scope

Zeroization is applied to API keys and channel tokens, but:

- Other sensitive data (agent prompts, conversation history) may contain sensitive information
- `Zeroizing<String>` only works for String types -- larger structs may still hold secrets in non-zeroized fields
- Memory pages may be swapped to disk

### 5. OFP Shared Secret Management

The OFP protocol requires a pre-shared secret but:

- The README does not specify how to generate or rotate this secret
- If the secret is compromised, all P2P communication is authenticated as the attacker
- No mention of secret rotation mechanism

### 6. WhatsApp Gateway Separation

The WhatsApp gateway runs as a separate Node.js process (`packages/whatsapp-gateway`):

- Not part of the main Rust binary
- Communicates via HTTP to port 3009
- Security of the Node.js gateway is separate from OpenFang's Rust security model
- The gateway stores WhatsApp session data outside OpenFang's credential vault

### 7. Skills Ecosystem Supply Chain

Skills from FangHub are scanned for prompt injection and SHA256 checksum verified, but:

- A compromised FangHub account could publish malicious skills
- SHA256 checksums are only as secure as the FangHub infrastructure
- No mention of FangHub security practices or vulnerability disclosure program

## Security Dependencies

| Dependency | Purpose |
|------------|---------|
| `sha2` | SHA-256 hashing (audit trail, checksums) |
| `hmac` | HMAC-SHA256 for OFP authentication |
| `subtle` | Constant-time comparison |
| `ed25519-dalek` | Ed25519 signing/verification |
| `zeroize` | Secret memory wiping |
| `governor` | GCRA rate limiting |
| `wasmtime` | WASM sandbox with metering |

## Reporting Security Issues

**Contact:** jaber@rightnowai.co (not GitHub issues)

**Scope includes:**
- Authentication/authorization bypass
- Remote code execution
- Path traversal
- SSRF
- Privilege escalation
- Information disclosure
- DoS via resource exhaustion
- Supply chain attacks via skills
- WASM sandbox escapes

**Response timeline:**
- Acknowledgment: 48 hours
- Initial assessment: 7 days
- Fix communicated: 14 days
