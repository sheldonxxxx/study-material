# Security and Performance Analysis: Moltis

## Overview

Moltis is a Rust-based personal AI gateway (monorepo with 52 crates) that handles chat channels, LLM providers, and tool execution. This analysis covers the security and performance architecture across the codebase.

---

## Security Patterns

### 1. Secrets Management

**Encryption at Rest (`crates/vault`)**
- XChaCha20-Poly1305 cipher for symmetric encryption
- DEK (Data Encryption Key) wrapped with password-derived KEK via Argon2id
- Vault has three states: Uninitialized, Sealed, Unsealed
- Recovery key support with separate wrapping
- AAD (Additional Authenticated Data) binds encrypted data to context (`"env:KEY_NAME"`)

```rust
// From vault.rs:DEK held in memory behind RwLock, None = sealed
pub struct Vault<C: Cipher = XChaCha20Poly1305Cipher> {
    dek: RwLock<Option<Zeroizing<[u8; 32]>>>,
}
```

- Credential store uses vault when available, stores env vars encrypted at rest

**Secrets Handling (`secrecy` crate)**
- Uses `secrecy::Secret<String>` wrapper for passwords, keys, tokens
- `expose_secret()` only at consumption point
- Manual Debug impl with `[REDACTED]` to prevent accidental logging

**API Key Security**
- Raw keys only shown once at creation time (`mk_<token>` format)
- Only SHA-256 hash stored in database
- Prefix (`mk_` + 8 chars) shown in UI for identification
- Scoped API keys with explicit permissions

**Session Tokens**
- Cryptographically random tokens (32 bytes, URL-safe base64)
- 30-day expiry
- Password changes invalidate all sessions

### 2. WebAuthn / Passkeys (`crates/auth/src/webauthn.rs`)

- Uses `webauthn-rs` v0.5 for passkey support
- Challenge TTL of 5 minutes prevents replay attacks
- In-flight challenges stored in `DashMap` with TTL-based expiry
- Host normalization strips ports and lowercases for consistent matching
- Supports multiple origins (localhost + mDNS hostnames)
- Registrations exclude existing credentials to prevent clone attacks

```rust
const CHALLENGE_TTL_SECS: u64 = 300; // 5 minutes

pub struct WebAuthnState {
    pending_registrations: DashMap<String, PendingRegistration>,
    pending_authentications: DashMap<String, PendingAuthentication>,
}
```

### 3. SSRF Protection (`crates/tools/src/ssrf.rs`)

Blocks private/loopback/link-local IPs for outbound fetches:
- IPv4: loopback, private (10.x, 172.16-31.x, 192.168.x), link-local, broadcast, unspecified, CGNAT (100.64.0.0/10)
- IPv6: loopback, unspecified, unique local (fc00::/7), link-local (fe80::/10)
- CIDR allowlist support for internal networks (e.g., `172.22.0.0/16`)
- Async and blocking variants for different call paths

```rust
pub fn is_private_ip(ip: &IpAddr) -> bool {
    match ip {
        IpAddr::V4(v4) => v4.is_loopback() || v4.is_private() || v4.is_link_local() || ...,
        IpAddr::V6(v6) => v6.is_loopback() || ...,
    }
}
```

### 4. Rate Limiting (`crates/httpd/src/request_throttle.rs`)

Token bucket per IP per scope:
- Login: 5 req/min (brute-force protection)
- Auth API: 120 req/min
- General API: 180 req/min
- Share links: 90 req/min
- WebSocket: 30 req/min (reconnect storm protection)

Cleanup every 512 requests to prevent memory growth.

```rust
struct ThrottleLimits {
    login: RateLimit { max_requests: 5, window: Duration::from_secs(60) },
    // ...
}
```

### 5. SQL Injection Prevention

**All SQL uses parameterized queries via sqlx:**
- No string interpolation in SQL
- All user values bound via `.bind()`
- Examples from `credential_store.rs`:

```rust
sqlx::query("SELECT password_hash FROM auth_password WHERE id = 1")
sqlx::query("DELETE FROM auth_sessions WHERE token = ?").bind(token)
sqlx::query_as("SELECT id, scopes FROM api_keys WHERE key_hash = ? AND revoked_at IS NULL")
```

### 6. WASM Sandboxing (`crates/tools/src/sandbox/wasm.rs`)

**Multi-layer isolation:**
1. **Fuel metering**: Default 1 billion instructions, configurable per tool
2. **Epoch interruption**: 100ms interval, external thread increments epoch for timeout
3. **Memory limits**: Default 16MB, per-tool overrides (calc: 2MB, web_fetch: 32MB)
4. **Filesystem isolation**: Only `/home/sandbox` and `/tmp` preopened; path traversal prevented via canonicalization checks
5. **Process limits**: ResourceLimiter trait implementation

```rust
// From wasm_engine.rs
wasm_config.consume_fuel(true);
wasm_config.epoch_interruption(true);
wasm_config.memory_reservation(bytes); // optional memory cap
```

**WASM built-in commands** (~20 coreutils): echo, cat, ls, mkdir, rm, cp, mv, pwd, env, head, tail, wc, sort, touch, which, true, false, test, [, basename, dirname

### 7. Input Validation

**TOML Config Schema Validation (`crates/config/src/validate.rs`):**
- Unknown field detection with Levenshtein-based suggestions
- Type checking via serde deserialization
- Semantic warnings for security issues:
  - `auth.disabled` + non-localhost bind
  - TLS disabled + non-localhost bind
  - Sandbox mode off
  - SSRF allowlist entries not valid CIDR

**Tool parameter extraction (`crates/tools/src/params.rs`):**
- `str_param()`: Trimmed, non-empty string extraction
- `require_str()`: Required params with error messages
- `u64_param()` / `bool_param()`: Typed extraction with defaults

### 8. WebSocket Security

- Origin validation rejects cross-origin WS upgrades (403)
- Loopback variants treated as equivalent
- Auth middleware for all `/api/*` except `/api/auth/*` and `/api/gon`

### 9. Docker Sandbox (`crates/tools/src/sandbox/docker.rs`)

- Container-based isolation for browser and exec tools
- Ephemeral containers with auto-cleanup
- Container socket access via Docker socket mount
- Optional network isolation (blocked/trusted/bypass)

### 10. Browser Sandboxing (`crates/browser/src/pool.rs`)

- Memory-based pool limits
- Hard TTL (30 min) prevents Chromium memory leak accumulation
- Idle timeout cleanup
- Session isolation per browser instance
- Chrome security flags: `--no-sandbox`, `--disable-setuid-sandbox` (for headless)

---

## Performance Patterns

### 1. Connection Pooling

**SQLite via sqlx:**
```rust
SqlitePool::connect(&db_url).await?
```
Multiple pools across crates (sessions, gateway, cron, projects, etc.)

**HTTP client (reqwest):**
```rust
reqwest::Client::builder()
    .timeout(Duration::from_secs(30))
    .build()?
```

### 2. Caching

**WASM Component Cache (`crates/tools/src/wasm_engine.rs`):**
- SHA-256 keyed HashMap cache for compiled WASM components
- Thread-safe via RwLock
- Pre-compiled `.cwasm` AOT deserialization support

```rust
pub struct WasmComponentEngine {
    engine: wasmtime::Engine,
    component_cache: RwLock<HashMap<[u8; 32], wasmtime::component::Component>>,
}
```

**DashMap for concurrent caches:**
- WebAuthn challenge state
- Request throttle buckets
- Session state stores

### 3. Lazy Tool Loading (`crates/agents/src/lazy_tools.rs`)

When `registry_mode = "lazy"`:
- Model discovers tools via `tool_search(query="...")`
- Activates tools on-demand by name
- Maximum 15 search results
- Avoids loading all tool schemas upfront

```rust
const MAX_SEARCH_RESULTS: usize = 15;

pub struct ToolSearchTool {
    full_registry: Arc<ToolRegistry>,
    activated: ActivatedTools,
}
```

### 4. Resource Management

**Browser Pool (`crates/browser/src/pool.rs`):**
- Instance reuse per session
- Memory-based admission control
- Automatic cleanup of idle/expired instances

```rust
pub struct BrowserPool {
    instances: RwLock<HashMap<String, Arc<Mutex<BrowserInstance>>>>,
    active_count: AtomicUsize,  // for metrics
}
```

**WASM Resource Limiter:**
```rust
impl wasmtime::ResourceLimiter for WasmResourceLimiter {
    fn memory_growing(&mut self, current: usize, desired: usize, maximum: Option<usize>) -> anyhow::Result<bool> {
        let within_limit = desired <= self.max_memory_bytes;
        let within_wasm_maximum = maximum.is_none_or(|max| desired <= max);
        Ok(within_limit && within_wasm_maximum)
    }
}
```

### 5. Async Patterns

**tokio-based async runtime throughout:**
- Full-featured tokio for async execution
- `spawn_blocking` for CPU-bound/blocking operations
- `tokio::join!` / `futures::join_all` for concurrent operations

**Thread pool for blocking operations:**
```rust
tokio::task::spawn_blocking(move || {
    // Container CLI checks, image pulls
})
```

### 6. Memory Management

- `tikv-jemallocator` for reduced memory fragmentation
- `Zeroizing<[u8; 32]>` for sensitive data (DEK)
- Explicit output truncation to prevent memory exhaustion

### 7. Database Optimization

- Migrations managed per-crate via sqlx
- Workspace-local migrations
- Index hints where applicable

---

## Security Testing

### Unit Tests
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

### Notable Absences
- No dedicated fuzzing suite found
- No property-based testing (proptest)
- No security-focused penetration testing tools

---

## Compliance and Code Quality

**Rust linting:**
- `clippy.toml`: Denies `expect_used`, `unwrap_used`
- Warnings treated as errors
- Pinned nightly toolchain

**TOML formatting:**
- taplo for TOML validation

**JavaScript:**
- Biome for JS/TS linting

**Secrets:**
- Pre-commit hooks prevent accidental secret commits
- `git reset HEAD~1` + re-commit workflow for accidents

---

## Summary Table

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
| Connection pooling | sqlx SqlitePool | Multiple crates |
| Caching | DashMap + RwLock for WASM | `crates/tools/src/wasm_engine.rs` |
| Resource limits | Browser pool + WASM limiter | `crates/browser`, `crates/tools/src/wasm_limits.rs` |
