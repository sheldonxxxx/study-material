# Security Patterns

## Overview

Moltis implements defense-in-depth with multiple security layers: encrypted secrets storage, WebAuthn authentication, WASM sandboxing, SSRF protection, and rate limiting.

---

## Secrets Management

### Encryption at Rest (`crates/vault`)

**XChaCha20-Poly1305 cipher** for symmetric encryption:

```rust
pub struct Vault<C: Cipher = XChaCha20Poly1305Cipher> {
    dek: RwLock<Option<Zeroizing<[u8; 32]>>>,
}
```

**Key derivation:**
- DEK (Data Encryption Key) wrapped with password-derived KEK via Argon2id
- Recovery key support with separate wrapping
- AAD (Additional Authenticated Data) binds encrypted data to context (`"env:KEY_NAME"`)

**Vault states:** Uninitialized -> Sealed -> Unsealed

### Secrets Handling (`secrecy` Crate)

```rust
use secrecy::{ExposeSecret, Secret};

// Use Secret<String> for all passwords, keys, tokens
pub struct ProviderConfig {
    pub api_key: Option<Secret<String>>,
}

// expose_secret() only at consumption point
// Custom Debug impl with [REDACTED] prevents accidental logging
```

### API Key Security

- Raw keys shown **only once** at creation (`mk_<token>` format)
- Only **SHA-256 hash** stored in database
- Prefix (`mk_` + 8 chars) shown in UI for identification
- Scoped API keys with explicit permissions

### Session Tokens

- Cryptographically random tokens (32 bytes, URL-safe base64)
- 30-day expiry
- Password changes invalidate all sessions

---

## WebAuthn / Passkeys

**`crates/auth/src/webauthn.rs`**

Uses `webauthn-rs` v0.5 for passkey support:

```rust
const CHALLENGE_TTL_SECS: u64 = 300; // 5 minutes

pub struct WebAuthnState {
    pending_registrations: DashMap<String, PendingRegistration>,
    pending_authentications: DashMap<String, PendingAuthentication>,
}
```

**Security measures:**
- Challenge TTL of 5 minutes prevents replay attacks
- In-flight challenges stored in `DashMap` with TTL-based expiry
- Host normalization strips ports and lowercases for consistent matching
- Supports multiple origins (localhost + mDNS hostnames)
- Registrations exclude existing credentials to prevent clone attacks

---

## SSRF Protection

**`crates/tools/src/ssrf.rs`**

Blocks private/loopback/link-local IPs for outbound fetches:

```rust
pub fn is_private_ip(ip: &IpAddr) -> bool {
    match ip {
        IpAddr::V4(v4) => v4.is_loopback() || v4.is_private() || v4.is_link_local() || ...,
        IpAddr::V6(v6) => v6.is_loopback() || ...,
    }
}
```

**Blocked ranges:**
- IPv4: loopback, private (10.x, 172.16-31.x, 192.168.x), link-local, broadcast, unspecified, CGNAT (100.64.0.0/10)
- IPv6: loopback, unspecified, unique local (fc00::/7), link-local (fe80::/10)

**CIDR allowlist support** for internal networks (e.g., `172.22.0.0/16`)

---

## Rate Limiting

**`crates/httpd/src/request_throttle.rs`**

Token bucket per IP per scope:

| Scope | Rate |
|-------|------|
| Login | 5 req/min (brute-force protection) |
| Auth API | 120 req/min |
| General API | 180 req/min |
| Share links | 90 req/min |
| WebSocket | 30 req/min (reconnect storm protection) |

Cleanup every 512 requests to prevent memory growth.

---

## SQL Injection Prevention

**All SQL uses parameterized queries via sqlx:**

```rust
// No string interpolation in SQL
// All user values bound via .bind()

sqlx::query("SELECT password_hash FROM auth_password WHERE id = 1")
sqlx::query("DELETE FROM auth_sessions WHERE token = ?").bind(token)
sqlx::query_as("SELECT id, scopes FROM api_keys WHERE key_hash = ? AND revoked_at IS NULL")
```

---

## WASM Sandboxing

**`crates/tools/src/sandbox/wasm.rs`**

Multi-layer isolation:

| Layer | Default | Per-Tool Override |
|-------|---------|-------------------|
| Fuel metering | 1 billion instructions | Configurable |
| Epoch interruption | 100ms interval | - |
| Memory limits | 16MB | calc: 2MB, web_fetch: 32MB |

```rust
// From wasm_engine.rs
wasm_config.consume_fuel(true);
wasm_config.epoch_interruption(true);
wasm_config.memory_reservation(bytes);
```

**Filesystem isolation:** Only `/home/sandbox` and `/tmp` preopened; path traversal prevented via canonicalization checks

**WASM built-in commands:** echo, cat, ls, mkdir, rm, cp, mv, pwd, env, head, tail, wc, sort, touch, which, true, false, test, [, basename, dirname (~20 coreutils)

---

## Docker Sandbox

**`crates/tools/src/sandbox/docker.rs`**

- Container-based isolation for browser and exec tools
- Ephemeral containers with auto-cleanup
- Container socket access via Docker socket mount
- Optional network isolation (blocked/trusted/bypass)

---

## Browser Sandboxing

**`crates/browser/src/pool.rs`**

- Memory-based pool limits
- Hard TTL (30 min) prevents Chromium memory leak accumulation
- Idle timeout cleanup
- Session isolation per browser instance
- Chrome security flags: `--no-sandbox`, `--disable-setuid-sandbox`

---

## Input Validation

### TOML Config Schema Validation

**`crates/config/src/validate.rs`:**
- Unknown field detection with Levenshtein-based suggestions
- Type checking via serde deserialization
- Semantic warnings for security issues:
  - `auth.disabled` + non-localhost bind
  - TLS disabled + non-localhost bind
  - Sandbox mode off
  - SSRF allowlist entries not valid CIDR

### Tool Parameter Extraction

**`crates/tools/src/params.rs`:**
```rust
str_param()      // Trimmed, non-empty string extraction
require_str()    // Required params with error messages
u64_param()      // Typed extraction with defaults
bool_param()      // Typed extraction with defaults
```

---

## WebSocket Security

- Origin validation rejects cross-origin WS upgrades (403)
- Loopback variants treated as equivalent
- Auth middleware for all `/api/*` except `/api/auth/*` and `/api/gon`

---

## Security Summary Table

| Pattern | Implementation | Location |
|---------|----------------|----------|
| Secrets at rest | XChaCha20-Poly1305 + Argon2id | `crates/vault` |
| WebAuthn | webauthn-rs with challenge TTL | `crates/auth` |
| SSRF protection | IP blocklist + CIDR allowlist | `crates/tools/src/ssrf.rs` |
| Rate limiting | Token bucket per IP/scope | `crates/httpd/src/request_throttle.rs` |
| SQL injection | Parameterized queries only | All sqlx usage |
| WASM sandbox | Fuel + epoch + memory limits | `crates/tools/src/sandbox/wasm.rs` |
| Input validation | Schema validation + param helpers | `crates/config`, `crates/tools/src/params.rs` |
| API keys | SHA-256 hash, prefix for ID | `crates/auth/src/credential_store.rs` |
| Sessions | 30-day expiry, invalidate on password change | `crates/auth` |
| Browser sandbox | Memory pool + TTL + security flags | `crates/browser/src/pool.rs` |

---

## Testing

### Security Unit Tests

- Password hashing/verification tests
- Session validation tests
- WebAuthn challenge expiry tests
- SSRF blocking/allowlist tests
- Rate limit throttle tests
- Tool parameter extraction tests
- Schema validation tests (100+ test cases)
- WASM resource limiter tests

### Integration Tests

- E2E tests with Playwright for auth flows
- WebSocket connection tests
- Channel integration tests

---

## Compliance

- **Rust linting:** `clippy.toml` denies `expect_used`, `unwrap_used`
- **TOML formatting:** taplo for TOML validation
- **JS linting:** Biome for JavaScript/TypeScript
- **Pre-commit hooks** prevent accidental secret commits
