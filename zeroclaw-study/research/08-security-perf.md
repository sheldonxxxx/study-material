# ZeroClaw Security and Performance Patterns

## Security Architecture

### 1. Secret Management

**Location:** `src/security/secrets.rs`

ZeroClaw implements defense-in-depth encrypted secret storage using ChaCha20-Poly1305 AEAD:

- **Encryption scheme:** 256-bit key stored in `~/.zeroclaw/.secret_key` with OS-level permissions (0600 on Unix, icacls on Windows)
- **Key generation:** Uses `ChaCha20Poly1305::generate_key(&mut OsRng)` for full 256-bit CSPRNG entropy (not UUID-based)
- **Nonce:** Fresh 12-byte random nonce per encryption, prepended to ciphertext
- **Storage format:** `enc2:<hex(nonce || ciphertext || tag)>` for new secrets
- **Legacy migration:** `enc:` prefix indicates XOR cipher (backward compatible, auto-migrates to `enc2:` on decrypt)
- **Platform permissions:** Unix uses `fs::Permissions::from_mode(0o600)`; Windows uses `takeown` then `icacls /inheritance:r /grant:r` (fixes issue #4532)

Key excerpt:
```rust
pub fn encrypt(&self, plaintext: &str) -> Result<String> {
    let key_bytes = self.load_or_create_key()?;
    let cipher = ChaCha20Poly1305::new(Key::from_slice(&key_bytes));
    let nonce = ChaCha20Poly1305::generate_nonce(&mut OsRng);
    let ciphertext = cipher.encrypt(&nonce, plaintext.as_bytes())?;
    // Prepend nonce to ciphertext for storage
}
```

**File permissions test coverage:** Extensive (tamper detection, wrong key rejection, truncate detection, Windows icacls ordering)

---

### 2. Authentication - Pairing Guard

**Location:** `src/security/pairing.rs`

First-connect authentication via one-time pairing codes:

- **Pairing code:** 6-digit numeric via rejection sampling from UUID v4 CSPRNG (eliminates modulo bias)
- **Bearer tokens:** 256-bit entropy via `rand::random()` with `zc_` prefix, stored as SHA-256 hashes
- **Brute force protection:** 5 attempts max, then 5-minute lockout per client ID
- **Memory bounds:** `MAX_TRACKED_CLIENTS = 10,000` with LRU eviction and stale entry pruning
- **Constant-time comparison:** `constant_time_eq()` uses bitwise `&` instead of logical `&&` to prevent timing attacks
- **Timing attack mitigation:**
```rust
// Track length mismatch as a usize (non-zero = different lengths)
let len_diff = a.len() ^ b.len();
// XOR each byte, padding the shorter input with zeros
let max_len = a.len().max(b.len());
let mut byte_diff = 0u8;
for i in 0..max_len {
    let x = *a.get(i).unwrap_or(&0);
    let y = *b.get(i).unwrap_or(&0);
    byte_diff |= x ^ y;
}
// Bitwise & (not &&) ensures constant-time execution
(len_diff == 0) & (byte_diff == 0)
```

---

### 3. WebAuthn / FIDO2 Hardware Key Authentication

**Location:** `src/security/webauthn.rs`

Minimal WebAuthn implementation (avoids heavy third-party libraries):

- **Algorithm:** ES256 (ECDSA P-256 + SHA-256) via `ring` crate
- **Challenge:** 32 bytes (256 bits) from `ring::rand::SystemRandom`
- **Credential storage:** JSON, encrypted via `SecretStore`, persisted to `webauthn_credentials.json`
- **Clone detection:** Sign counter per credential, monotonic increase required
- **Signature verification:** Uses `ring::signature::ECDSA_P256_SHA256_ASN1`
- **Attestation parsing:** Supports uncompressed P-256 point (0x04 || x || y, 65 bytes) and simplified JSON format

---

### 4. Security Policy - Command/Path Validation

**Location:** `src/security/policy.rs`

Layered command execution policy with five independent validation gates:

1. **Subshell blocking:** `` ` ``, `$(`, `${`, `<(`, `>(`
2. **Redirection blocking:** `<`, `>`, `>>` (unquoted)
3. **Tee blocking:** `tee` command (can bypass redirects)
4. **Background chain blocking:** Single `&` (but `&&` allowed)
5. **Per-segment allowlist:** Validates each pipe segment separately

**Risk classification:**
- High: `rm`, `curl`, `wget`, `sudo`, `shutdown`, `reboot`, `powershell`, etc.
- Medium: `git commit/push/reset`, `npm install`, `cargo install`, `touch`, `mkdir`, `mv`, `cp`
- Low: read-only commands

**Path validation layers:**
- Null-byte injection prevention
- `..` path traversal detection via `Path::components()`
- URL-encoded traversal (`%2f..`)
- Tilde expansion (`~user` blocked, `~/` allowed)
- Workspace confinement with `allowed_roots` exceptions
- Symlink escape prevention via canonicalization

```rust
pub fn is_resolved_path_allowed(&self, resolved: &Path) -> bool {
    // Check canonicalized workspace root
    let workspace_root = self.workspace_dir.canonicalize().unwrap_or_else(|_| self.workspace_dir.clone());
    if resolved.starts_with(&workspace_root) { return true; }
    // Check allowed_roots before forbidden checks
    for root in &self.allowed_roots { ... }
    // Forbidden prefix match
}
```

---

### 5. Credential Leak Detection

**Location:** `src/security/leak_detector.rs`

Outbound content scanning for credential exfiltration:

- **API key patterns:** Stripe, OpenAI, Anthropic, Google, GitHub, generic `api_key=`
- **AWS credentials:** `AKIA*` access keys, `aws_secret_access_key` patterns
- **Private keys:** PEM-encoded RSA/EC/OPENSSH private keys
- **JWT tokens:** `eyJ*.eyJ*.*` base64url pattern
- **Database URLs:** PostgreSQL, MySQL, MongoDB, Redis connection strings
- **High-entropy detection:** Shannon entropy threshold (4.375 at 0.7 sensitivity) with URL/media marker stripping
- **False positive handling:** Strips URLs before entropy check, media markers `[IMAGE:...]` excluded

---

### 6. Prompt Injection Defense

**Location:** `src/security/prompt_guard.rs`

Multi-pattern prompt injection detection with configurable sensitivity:

- **System override:** `ignore previous instructions`, `disregard all`, `new system prompt`
- **Role confusion:** `you are now`, `act as`, `from now on you are`
- **Tool JSON injection:** `tool_calls`, `function_call`, JSON escape attempts
- **Secret extraction:** `show me all secrets`, `list passwords`, `dump vault`
- **Command injection:** backticks, `$(`, `&&`, `||`, `;`, `|`, `>/dev/`, `2>&1`
- **Jailbreak patterns:** DAN mode, developer mode, hypothetical framing, base64 decode tricks
- **Scoring:** Normalized 0.0-1.0 across 6 categories (max 6.0 raw)
- **Actions:** Warn (default), Block, Sanitize

---

### 7. Sandboxing

**Location:** `src/security/traits.rs`, `src/security/landlock.rs`, `src/security/bubblewrap.rs`, `src/security/firejail.rs`

Pluggable sandbox architecture via `Sandbox` trait:

```rust
pub trait Sandbox: Send + Sync {
    fn wrap_command(&self, cmd: &mut Command) -> std::io::Result<()>;
    fn is_available(&self) -> bool;
    fn name(&self) -> &str;
    fn description(&self) -> str;
}
```

**Backends (auto-selected at runtime):**
- **Landlock** (Linux 5.13+): Filesystem access control via Rust `landlock` crate
- **Bubblewrap:** Unprivileged container (Linux)
- **Firejail:** SUID sandbox (Linux)
- **Seatbelt:** macOS sandbox
- **Docker:** Container-based isolation
- **NoopSandbox:** Fallback (application-layer only)

**Landlock ruleset:** Workspace (read/write), /tmp (read/write), /usr and /bin (read-only)

---

### 8. Audit Logging

**Location:** `src/security/audit.rs`

Tamper-evident audit trail with Merkle hash chain:

- **Chain structure:** `entry_hash = SHA-256(prev_hash || canonical_json)`
- **Genesis seed:** `0000000000000000000000000000000000000000000000000000000000000000`
- **Event types:** CommandExecution, FileAccess, ConfigChange, AuthSuccess, AuthFailure, PolicyViolation, SecurityEvent
- **Optional HMAC-SHA256 signing:** Requires `ZEROCLAW_AUDIT_SIGNING_KEY` env var (32 bytes hex)
- **Log rotation:** Configurable max size with numbered backups
- **Recovery:** Reads last entry on startup to continue chain
- **Verification:** `verify_chain()` checks sequence continuity, hash linkage, and signatures

```rust
fn compute_entry_hash(prev_hash: &str, event: &AuditEvent) -> String {
    let content = serde_json::json!({
        "timestamp": event.timestamp,
        "event_id": event.event_id,
        "event_type": event.event_type,
        // ... excluding chain fields
        "sequence": event.sequence,
    });
    let mut hasher = Sha256::new();
    hasher.update(prev_hash.as_bytes());
    hasher.update(content_json.as_bytes());
    hex::encode(hasher.finalize())
}
```

---

### 9. Observability / Security Monitoring

**Location:** `src/observability/mod.rs`

Pluggable observability backend factory:

| Backend | Feature Flag | Purpose |
|---------|-------------|---------|
| `noop` | (default) | No-op observer |
| `log` | (built-in) | Structured logging via `tracing` |
| `verbose` | (built-in) | Extended logging |
| `prometheus` | `observability-prometheus` | Prometheus metrics |
| `otel` | `observability-otel` | OpenTelemetry tracing + metrics |

Factory pattern with graceful fallback when features disabled.

---

## Performance Patterns

### 1. Multi-Stage Memory Retrieval Pipeline

**Location:** `src/memory/retrieval.rs`

Three-stage retrieval with early-return optimization:

```
cache (LRU, 256 entries, 5min TTL)
    ↓ miss
fts (FTS5 keyword search)
    ↓ if top_score >= 0.85 (early return)
vector (vector similarity + hybrid merge)
```

- **Hot cache:** In-memory `HashMap<String, CachedResult>` with LRU eviction
- **FTS early return:** If FTS score >= 0.85, skip vector stage entirely
- **Cache invalidation:** Explicit `invalidate_cache()` call after store operations
- **Configurable stages:** `retrieval_stages` config option

---

### 2. Response Cache (Two-Tier LLM Response Caching)

**Location:** `src/memory/response_cache.rs`

Avoids burning tokens on repeated prompts:

- **Key:** `SHA-256(model || system_prompt || user_prompt)` - 64 hex chars
- **Two-tier architecture:**
  - Hot (in-memory): 256-entry LRU, TTL-based expiration
  - Warm (SQLite): Persistent, WAL mode, indexed by `accessed_at`
- **SQLite optimizations:**
  ```sql
  PRAGMA journal_mode = WAL;
  PRAGMA synchronous = NORMAL;
  PRAGMA temp_store = MEMORY;
  ```
- **Eviction:** TTL expiration + LRU when over `max_entries`
- **Statistics:** Tracks total entries, hits, tokens saved (token_count × hit_count)
- **Concurrency:** Thread-safe with `parking_lot::Mutex`, handles concurrent reads

---

### 3. SQLite Persistence Patterns

**Locations:** `src/memory/sqlite.rs`, various memory backends

Consistent SQLite patterns across the codebase:

- **WAL mode** for concurrent read/write
- **Normal sync** for balance of safety and speed
- **Memory temp store** for temporary structures
- **Indexed queries:** `CREATE INDEX IF NOT EXISTS idx_xxx ON table(column)`
- **Parameterized queries** via `rusqlite::params!` macro

---

### 4. Memory Budgets and Limits

**Hard limits found:**
- `MAX_TRACKED_CLIENTS = 10,000` (pairing guard failed attempts map)
- `FAILED_ATTEMPT_RETENTION_SECS = 900` (15 minutes)
- `MAX_PAIR_ATTEMPTS = 5` before lockout
- Response cache hot entries: configurable (default 256)
- Response cache SQLite entries: configurable (default unlimited)

---

### 5. Async Concurrency

**Pattern:** `tokio::task::spawn_blocking` for CPU-bound or blocking operations:
```rust
// PairingGuard.try_pair_async
let handle = tokio::task::spawn_blocking(move || this.try_pair_blocking(&code, &client_id));
```

---

## Security Summary

| Category | Implementation | Strength |
|----------|---------------|----------|
| Secrets at rest | ChaCha20-Poly1305 AEAD | Strong |
| Secrets in transit | TLS via `rustls` | Strong |
| AuthN (pairing) | One-time codes + SHA-256 tokens | Strong |
| AuthN (WebAuthn) | ES256 + clone detection | Strong |
| AuthZ | Command allowlist + risk tiers + path confinement | Strong |
| Input validation | Regex patterns + shell parsing | Medium-Strong |
| Prompt injection | Multi-pattern scanner with scoring | Medium |
| Credential leak | High-entropy detection + API key patterns | Strong |
| Audit | Merkle hash chain + optional HMAC | Strong |
| Sandboxing | Landlock/Firejail/Bubblewrap (platform-dependent) | Medium-Strong |
| Observability | Pluggable: log/prometheus/otel | Flexible |

## Performance Summary

| Category | Pattern | Notes |
|----------|---------|-------|
| Memory retrieval | 3-stage pipeline | FTS early-return optimization |
| LLM response cache | Two-tier (hot + warm) | Avoids repeated API calls |
| SQLite | WAL + Normal sync | Concurrent reads |
| Async | Tokio + spawn_blocking | Non-blocking I/O |
| Memory bounds | Configurable limits | Prevents unbounded growth |

---

## Key Files

| File | Purpose |
|------|---------|
| `src/security/mod.rs` | Security subsystem entry point |
| `src/security/secrets.rs` | ChaCha20-Poly1305 encrypted secret store |
| `src/security/pairing.rs` | First-connect authentication |
| `src/security/webauthn.rs` | FIDO2 hardware key auth |
| `src/security/policy.rs` | Command/path execution policy |
| `src/security/leak_detector.rs` | Credential exfiltration detection |
| `src/security/prompt_guard.rs` | Prompt injection defense |
| `src/security/audit.rs` | Tamper-evident audit logging |
| `src/security/traits.rs` | Sandbox trait definition |
| `src/security/landlock.rs` | Linux Landlock sandbox |
| `src/memory/retrieval.rs` | Multi-stage retrieval pipeline |
| `src/memory/response_cache.rs` | Two-tier LLM response cache |
| `src/observability/mod.rs` | Observability factory |
