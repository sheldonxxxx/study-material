# openfang Security and Performance Patterns

## Overview

The openfang project implements defense-in-depth security across multiple layers, combined with performance optimizations for async Rust workloads. This analysis documents the key patterns found in the codebase.

---

## Security Patterns

### 1. Credential Storage — `crates/openfang-extensions/src/vault.rs`

**Encryption:**
- AES-256-GCM authenticated encryption (from `aes-gcm` crate)
- OFV1 magic bytes for vault file format versioning
- Random 96-bit nonce per encryption operation

**Key Derivation:**
- Argon2id (via `argon2` crate) for deriving encryption key from master key + random salt
- Master key: 256-bit random, generated via `OsRng`

**Key Storage:**
- Primary: OS keyring (Windows Credential Manager / macOS Keychain / Linux Secret Service API)
- Fallback: `OPENFANG_VAULT_KEY` environment variable
- Keyring uses machine-specific obfuscation (SHA-256 of machine fingerprint + service name XOR mask)

**Memory Protection:**
- All secrets stored as `Zeroizing<String>` — automatically zeroed on drop
- `Drop` impl clears `entries` HashMap and `cached_key`

```rust
// vault.rs: key derivation
fn derive_key(master_key: &[u8; 32], salt: &[u8]) -> ExtensionResult<Zeroizing<[u8; 32]>> {
    let mut derived = Zeroizing::new([0u8; 32]);
    Argon2::default()
        .hash_password_into(master_key, salt, derived.as_mut())?;
    Ok(derived)
}
```

---

### 2. OAuth2 PKCE — `crates/openfang-extensions/src/oauth.rs`

**PKCE Flow (RFC 7636):**
- Code verifier: 256-bit random via `OsRng`, base64url encoded
- Code challenge: SHA-256 hash of verifier, base64url encoded (S256 method)
- State parameter: 128-bit random for CSRF protection

**Token Storage:**
- Access/refresh tokens wrapped in `Zeroizing<String>`
- `OAuthTokens::access_token_zeroizing()` / `refresh_token_zeroizing()` helpers

**Callback Security:**
- Localhost server on random ephemeral port
- 5-minute timeout on entire OAuth flow
- State parameter validation before accepting authorization code

```rust
// oauth.rs: PKCE generation
fn generate_pkce() -> PkcePair {
    let mut bytes = [0u8; 32];
    rand::rngs::OsRng.fill_bytes(&mut bytes);
    let verifier = Zeroizing::new(base64_url_encode(&bytes));
    let challenge = {
        let mut hasher = Sha256::new();
        hasher.update(verifier.as_bytes());
        base64_url_encode(&hasher.finalize())
    };
    PkcePair { verifier, challenge }
}
```

---

### 3. WASM Sandbox — `crates/openfang-runtime/src/sandbox.rs`

**Dual Metering:**
1. **Fuel metering** (`config.consume_fuel(true)`) — deterministic CPU instruction counting
2. **Epoch interruption** — wall-clock timeout via watchdog thread

**Deny-by-Default Capabilities:**
- `GuestState` carries `Vec<Capability>` checked before every `host_call`
- No filesystem, network, or credential access unless explicitly granted
- Host functions isolated in `"openfang"` import module (not WASI)

**Memory Safety:**
- Input bounds check: `if end > mem_data.len()` before write
- Output bounds check: `if result_ptr + result_len > mem_data.len()`
- Result pointer unpacked from `i64`: `(result_ptr << 32) | result_len`

**Guest ABI:**
- Required exports: `memory`, `alloc(size: i32) -> i32`, `execute(input_ptr, input_len) -> i64`
- `execute` returns packed (pointer, length) for variable-length results

```rust
// sandbox.rs: fuel + epoch dual metering
store.set_fuel(config.fuel_limit).unwrap();
store.set_epoch_deadline(1);
let _watchdog = std::thread::spawn(move || {
    std::thread::sleep(std::time::Duration::from_secs(timeout));
    engine_clone.increment_epoch();
});
```

---

### 4. SSRF Protection — `crates/openfang-runtime/src/web_fetch.rs` + `host_functions.rs`

**DNS Resolution Check:**
- Hostname blocklist: `localhost`, `ip6-localhost`, `metadata.google.internal`, `metadata.aws.internal`, `instance-data`, `169.254.169.254`, `100.100.100.200`, `192.0.0.192`, `0.0.0.0`, `::1`
- Private IP check after DNS resolution:
  - IPv4: `10.x.x.x`, `172.16-31.x.x`, `192.168.x.x`, `169.254.x.x`
  - IPv6: `fc00::/7` (unique local), `fe80::/10` (link-local)

**Scheme Restriction:**
- Only `http://` and `https://` allowed (blocks `file://`, `ftp://`, `gopher://`)

**Response Size Limit:**
- Configurable `max_response_bytes` cap on `content-length`

```rust
// web_fetch.rs: SSRF check before any network I/O
pub(crate) fn check_ssrf(url: &str) -> Result<(), String> {
    // Only allow http/https
    if !url.starts_with("http://") && !url.starts_with("https://") {
        return Err("Only http:// and https:// URLs are allowed".to_string());
    }
    // Hostname blocklist + DNS resolution + private IP check
}
```

---

### 5. Rate Limiting — `crates/openfang-api/src/rate_limiter.rs`

**Algorithm:** GCRA (Generic Cell Rate Algorithm) via `governor` crate
- State: `DashMapStateStore<IpAddr>` — thread-safe, concurrent
- Quota: 500 tokens per minute per IP

**Cost-Aware Operations:**
| Operation | Cost |
|-----------|------|
| GET /api/health | 1 |
| GET /api/agents | 2 |
| POST /api/agents | 50 |
| POST /api/agents/*/message | 30 |
| POST /api/agents/*/run | 100 |
| POST /api/skills/install | 50 |
| POST /api/migrate | 100 |

**Response:** 429 Too Many Requests with `retry-after: 60` header

---

### 6. Provider Circuit Breaker — `crates/openfang-runtime/src/auth_cooldown.rs`

**States:** Closed (healthy) → Open (cooldown) → HalfOpen (probe)

**Exponential Backoff:**
- General errors: base 60s, multiplier 5.0, max 3600s
- Billing errors (402): base 18000s, multiplier 2.0, max 86400s (24h)
- Max exponent cap: 3 (general) / 10 (billing)

**Half-Open Probing:**
- After cooldown expires, allows ONE probe request through
- Probe interval: 30 seconds minimum between probes
- Success on probe → circuit closes; failure extends cooldown

**Auth Profile Rotation:**
- Per-profile cooldown tracking (`provider::profile` key)
- Sorted by priority, selects first non-cooldown profile
- Falls back to first profile if all in cooldown (least bad)

---

### 7. Taint Tracking — `crates/openfang-types/src/taint.rs`

**Labels:**
- `ExternalNetwork` — data from external network requests
- `UserInput` — direct user input
- `Pii` — personally identifiable information
- `Secret` — API keys, tokens, passwords
- `UntrustedAgent` — output from sandboxed agents

**Sinks:**
| Sink | Blocked Labels |
|------|---------------|
| `shell_exec` | ExternalNetwork, UntrustedAgent, UserInput |
| `net_fetch` | Secret, Pii |
| `agent_message` | Secret |

**Declassification:** Explicit `declassify()` call removes label — caller asserts sanitization

---

### 8. Loop Guard — `crates/openfang-runtime/src/loop_guard.rs`

**Detection Mechanisms:**
1. **Call-count hashing:** SHA-256(tool_name | params_json)
2. **Outcome-aware:** Tracks (call_hash + result_hash) — identical results escalate faster
3. **Ping-pong:** Detects A-B-A-B and A-B-C-A-B-C cycling patterns

**Thresholds:**
- Default warn: 3 calls, block: 5 calls
- Global circuit breaker: 30 total calls per loop
- Poll tools (shell_exec status checks): 3x multiplier

**Backoff Suggestions:**
- Schedule: [5000, 10000, 30000, 60000] ms for polling tools
- `get_poll_backoff()` suggests delay based on call count

---

### 9. Subprocess Sandbox — `crates/openfang-runtime/src/subprocess_sandbox.rs`

**Environment Stripping:**
- `env_clear()` removes all inherited env vars
- Allowlist re-adds: `PATH`, `HOME`, `TMPDIR`, `TMP`, `TEMP`, `LANG`, `LC_ALL`, `TERM`
- Windows add: `USERPROFILE`, `SYSTEMROOT`, `APPDATA`, `LOCALAPPDATA`, `COMSPEC`, `WINDIR`, `PATHEXT`

**Shell Metacharacter Blocking:**
- Command substitution: backtick `` ` ``, `$()`, `${}`
- Chaining: `;`, `|`, `&&`, `||`, `&`
- I/O redirection: `>`, `<`, `>>`, `<<<`, `<()`
- Expansion: `{`, `}`, newlines, null bytes

**Path Validation:**
- Rejects `..` components in executable paths
- Default safe_bins: `echo`, `cat`, `sort`

**Process Tree Kill:**
- SIGTERM graceful → wait grace period → SIGKILL force
- Cross-platform: Unix `kill` / Windows `taskkill`

---

### 10. Security Headers — `crates/openfang-api/src/middleware.rs`

```rust
// security_headers middleware
headers.insert("x-content-type-options", "nosniff");
headers.insert("x-frame-options", "DENY");
headers.insert("x-xss-protection", "1; mode=block");
headers.insert("content-security-policy", "default-src 'self'; script-src 'self' 'unsafe-inline' ...");
headers.insert("referrer-policy", "strict-origin-when-cross-origin");
headers.insert("cache-control", "no-store, no-cache, must-revalidate");
headers.insert("strict-transport-security", "max-age=63072000; includeSubDomains");
```

---

### 11. Authentication — `crates/openfang-api/src/middleware.rs`

**Token Comparison:**
- `subtle::ConstantTimeEq` for constant-time comparison (prevents timing attacks)
- Supports: `Authorization: Bearer <token>`, `X-API-Key` header, `?token=` query param, session cookie

**Public Endpoints (no auth required):**
- GET `/api/health`, `/api/status`, `/api/version`
- GET `/api/agents` (listing only; POST requires auth)
- GET `/api/config`, `/api/models`, `/api/providers`, `/api/budget`
- WebSocket SSE stream `/api/logs/stream`

**Shutdown Protection:**
- `/api/shutdown` restricted to loopback IP only

---

### 12. Path Traversal Prevention — `crates/openfang-runtime/src/host_functions.rs`

**Two-Phase Path Resolution:**
1. Reject any path with `ParentDir` component (`..`)
2. Canonicalize to resolve symlinks

```rust
// host_functions.rs: safe_resolve_path
fn safe_resolve_path(path: &str) -> Result<std::path::PathBuf, serde_json::Value> {
    let p = Path::new(path);
    for component in p.components() {
        if matches!(component, Component::ParentDir) {
            return Err(json!({"error": "Path traversal denied: '..' components forbidden"}));
        }
    }
    std::fs::canonicalize(p).map_err(...)
}
```

---

## Performance Patterns

### 1. Async Runtime — Tokio

**Full `tokio` runtime** with `full` feature set across all crates.

**Key Patterns:**
- `spawn_blocking` for CPU-bound work (WASM execution in sandbox.rs)
- `tokio::time::timeout` for deadline enforcement
- `tokio::select!` for multiplexing stdout/stderr/process-exit (subprocess_sandbox.rs)

```rust
// subprocess_sandbox.rs: async multiplexing
tokio::select! {
    result = stdout_read => { ... }
    result = stderr_read => { ... }
    result = child.wait() => { ... }
}
```

---

### 2. Web Cache — `crates/openfang-runtime/src/web_cache.rs`

**Implementation:** `DashMap<String, CacheEntry>` — concurrent, thread-safe

**Features:**
- TTL-based expiration (lazy eviction on `get()`)
- `Duration::ZERO` TTL = disabled (zero-cost passthrough)
- `evict_expired()` for bulk cleanup
- `retain()` for atomic expired entry removal

```rust
// web_cache.rs: TTL with lazy eviction
pub fn get(&self, key: &str) -> Option<String> {
    if self.ttl.is_zero() { return None; }
    let entry = self.entries.get(key)?;
    if entry.inserted_at.elapsed() > self.ttl {
        self.entries.remove(key);
        None
    } else {
        Some(entry.value.clone())
    }
}
```

---

### 3. GCRA Rate Limiter State — `governor` + `DashMap`

```rust
// rate_limiter.rs
pub type KeyedRateLimiter = RateLimiter<IpAddr, DashMapStateStore<IpAddr>, DefaultClock>;
Arc<RateLimiter::keyed(Quota::per_minute(NonZeroU32::new(500).unwrap()))>
```

---

### 4. Provider Cooldown State — `DashMap`

```rust
// auth_cooldown.rs
pub struct ProviderCooldown {
    config: CooldownConfig,
    states: DashMap<String, ProviderState>, // provider -> state
}
```

---

### 5. HTTP Client — `reqwest`

**Configuration:**
- `rustls` TLS backend (not native-tls)
- User-Agent set to identify as OpenFang
- Gzip, deflate, brotli automatic decompression
- Configurable timeout per request

```rust
// web_fetch.rs: HTTP client setup
let client = reqwest::Client::builder()
    .user_agent(crate::USER_AGENT)
    .timeout(std::time::Duration::from_secs(config.timeout_secs))
    .gzip(true).deflate(true).brotli(true)
    .build()
    .unwrap_or_default();
```

---

### 6. Build Profiles — `Cargo.toml`

| Profile | LTO | Codegen | Strip | Opt |
|---------|-----|---------|-------|-----|
| `release` | fat | 1 | true | 3 |
| `release-fast` | thin | 8 | false | 2 |

**Optimization for release:** `lto = "fat"`, `codegen-units = 1`, `strip = true` produces smaller, faster binaries at cost of compile time.

---

## Summary Table

| Pattern | Location | Purpose |
|---------|----------|---------|
| AES-256-GCM vault | `vault.rs` | Encrypted credential storage |
| Argon2id key derivation | `vault.rs` | Master key → encryption key |
| Zeroizing secrets | `vault.rs`, `oauth.rs` | Memory zeroing on drop |
| PKCE OAuth2 | `oauth.rs` | Safe OAuth flows without client secret |
| WASM dual metering | `sandbox.rs` | CPU + wall-clock resource limits |
| Capability gates | `host_functions.rs` | Deny-by-default permissions |
| SSRF protection | `web_fetch.rs`, `host_functions.rs` | Block private IP access |
| GCRA rate limiting | `rate_limiter.rs` | Per-IP request throttling |
| Circuit breaker | `auth_cooldown.rs` | Provider failure isolation |
| Taint tracking | `taint.rs` | Information flow control |
| Loop guard | `loop_guard.rs` | Infinite loop prevention |
| Subprocess sandbox | `subprocess_sandbox.rs` | Shell command isolation |
| Shell metachar blocking | `subprocess_sandbox.rs` | Command injection prevention |
| Security headers | `middleware.rs` | Browser-level protections |
| Constant-time auth | `middleware.rs` | Timing attack prevention |
| Path canonicalization | `host_functions.rs` | Traversal escape prevention |
| DashMap caching | `web_cache.rs` | Thread-safe in-memory cache |
| Tokio async | Throughout | Concurrent I/O |
| Release optimization | `Cargo.toml` | Build performance tuning |

---

## Observations

**Strengths:**
- Defense-in-depth: multiple independent security layers
- Memory safety: `Zeroizing` for secrets, bounds checks in WASM
- Protocol correctness: PKCE proper implementation, GCRA algorithm
- Comprehensive testing: extensive unit tests for security boundaries

**Potential Concerns:**
- Machine-specific keyring obfuscation (XOR with SHA-256 hash) is not true encryption — physical access to the machine could potentially recover the key. OS keyring is the strong path.
- `subprocess_sandbox.rs` exec policy defaults to Allowlist mode, but Full mode (`ExecSecurityMode::Full`) allows arbitrary commands with only a warning log — risky for production.
- OAuth callback server binds to `127.0.0.1` without TLS — safe for localhost but requires careful network isolation.
