# ZeroClaw Security Analysis

**Repository:** `/Users/sheldon/Documents/claw/reference/zeroclaw/`
**Study Path:** `/Users/sheldon/Documents/claw/zeroclaw-study/`
**Generated:** 2026-03-26

## Security Architecture Summary

ZeroClaw implements defense-in-depth security across multiple layers:

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

---

## 1. Secret Management

**Location:** `src/security/secrets.rs`

### Encryption Scheme

ChaCha20-Poly1305 AEAD with defense-in-depth:

- **Key storage**: 256-bit key in `~/.zeroclaw/.secret_key` with OS-level permissions
- **Unix**: `fs::Permissions::from_mode(0o600)`
- **Windows**: `takeown` then `icacls /inheritance:r /grant:r` (fixes issue #4532)
- **Key generation**: `ChaCha20Poly1305::generate_key(&mut OsRng)` - full 256-bit CSPRNG entropy
- **Nonce**: Fresh 12-byte random nonce per encryption, prepended to ciphertext

### Storage Format

```
enc2:<hex(nonce || ciphertext || tag)>
```

Legacy format (`enc:`) indicates XOR cipher with auto-migration to `enc2:` on decrypt.

### Key Excerpt

```rust
pub fn encrypt(&self, plaintext: &str) -> Result<String> {
    let key_bytes = self.load_or_create_key()?;
    let cipher = ChaCha20Poly1305::new(Key::from_slice(&key_bytes));
    let nonce = ChaCha20Poly1305::generate_nonce(&mut OsRng);
    let ciphertext = cipher.encrypt(&nonce, plaintext.as_bytes())?;
    // Prepend nonce to ciphertext for storage
}
```

**Test coverage**: Extensive (tamper detection, wrong key rejection, truncate detection, Windows icacls ordering)

---

## 2. Authentication - Pairing Guard

**Location:** `src/security/pairing.rs`

### First-Connect Authentication

One-time pairing codes with brute force protection:

- **Pairing code**: 6-digit numeric via rejection sampling from UUID v4 CSPRNG (eliminates modulo bias)
- **Bearer tokens**: 256-bit entropy via `rand::random()` with `zc_` prefix, stored as SHA-256 hashes
- **Brute force protection**: 5 attempts max, then 5-minute lockout per client ID
- **Memory bounds**: `MAX_TRACKED_CLIENTS = 10,000` with LRU eviction and stale entry pruning

### Timing Attack Mitigation

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

Uses bitwise `&` instead of logical `&&` to prevent timing attacks.

---

## 3. WebAuthn / FIDO2 Hardware Key Authentication

**Location:** `src/security/webauthn.rs`

Minimal implementation (avoids heavy third-party libraries):

- **Algorithm**: ES256 (ECDSA P-256 + SHA-256) via `ring` crate
- **Challenge**: 32 bytes (256 bits) from `ring::rand::SystemRandom`
- **Credential storage**: JSON, encrypted via `SecretStore`, persisted to `webauthn_credentials.json`
- **Clone detection**: Sign counter per credential, monotonic increase required
- **Signature verification**: Uses `ring::signature::ECDSA_P256_SHA256_ASN1`
- **Attestation parsing**: Supports uncompressed P-256 point (0x04 || x || y, 65 bytes)

---

## 4. Security Policy - Command/Path Validation

**Location:** `src/security/policy.rs`

### Five Independent Validation Gates

1. **Subshell blocking**: `` ` ``, `$(`, `${`, `<(`, `>(`
2. **Redirection blocking**: `<`, `>`, `>>` (unquoted)
3. **Tee blocking**: `tee` command (can bypass redirects)
4. **Background chain blocking**: Single `&` (but `&&` allowed)
5. **Per-segment allowlist**: Validates each pipe segment separately

### Risk Classification

| Tier | Commands |
|------|----------|
| High | `rm`, `curl`, `wget`, `sudo`, `shutdown`, `reboot`, `powershell`, etc. |
| Medium | `git commit/push/reset`, `npm install`, `cargo install`, `touch`, `mkdir`, `mv`, `cp` |
| Low | Read-only commands |

### Path Validation Layers

- Null-byte injection prevention
- `..` path traversal detection via `Path::components()`
- URL-encoded traversal (`%2f..`)
- Tilde expansion (`~user` blocked, `~/` allowed)
- Workspace confinement with `allowed_roots` exceptions
- Symlink escape prevention via canonicalization

---

## 5. Credential Leak Detection

**Location:** `src/security/leak_detector.rs`

Outbound content scanning for credential exfiltration:

### Detection Patterns

| Type | Pattern |
|------|---------|
| API keys | Stripe, OpenAI, Anthropic, Google, GitHub, generic `api_key=` |
| AWS credentials | `AKIA*` access keys, `aws_secret_access_key` |
| Private keys | PEM-encoded RSA/EC/OPENSSH private keys |
| JWT tokens | `eyJ*.eyJ*.*` base64url pattern |
| Database URLs | PostgreSQL, MySQL, MongoDB, Redis connection strings |
| High-entropy | Shannon entropy threshold (4.375 at 0.7 sensitivity) |

False positive handling: strips URLs before entropy check, media markers `[IMAGE:...]` excluded.

---

## 6. Prompt Injection Defense

**Location:** `src/security/prompt_guard.rs`

Multi-pattern prompt injection detection with configurable sensitivity:

### Detection Categories

- **System override**: `ignore previous instructions`, `disregard all`, `new system prompt`
- **Role confusion**: `you are now`, `act as`, `from now on you are`
- **Tool JSON injection**: `tool_calls`, `function_call`, JSON escape attempts
- **Secret extraction**: `show me all secrets`, `list passwords`, `dump vault`
- **Command injection**: backticks, `$(`, `&&`, `||`, `;`, `|`, `>/dev/`, `2>&1`
- **Jailbreak patterns**: DAN mode, developer mode, hypothetical framing, base64 decode tricks

### Scoring

- Normalized 0.0-1.0 across 6 categories (max 6.0 raw)
- Actions: Warn (default), Block, Sanitize

---

## 7. Sandboxing

**Location:** `src/security/traits.rs`, `src/security/landlock.rs`, `src/security/bubblewrap.rs`, `src/security/firejail.rs`

### Pluggable Sandbox Architecture

```rust
pub trait Sandbox: Send + Sync {
    fn wrap_command(&self, cmd: &mut Command) -> std::io::Result<()>;
    fn is_available(&self) -> bool;
    fn name(&self) -> &str;
    fn description(&self) -> str;
}
```

### Backend Selection (Auto-selected at Runtime)

| Backend | Platform | Notes |
|---------|----------|-------|
| Landlock | Linux 5.13+ | Filesystem access control via Rust `landlock` crate |
| Bubblewrap | Linux | Unprivileged container |
| Firejail | Linux | SUID sandbox |
| Seatbelt | macOS | Application sandbox |
| Docker | Any | Container-based isolation |
| NoopSandbox | Fallback | Application-layer only |

### Landlock Ruleset

Workspace (read/write), /tmp (read/write), /usr and /bin (read-only)

---

## 8. Audit Logging

**Location:** `src/security/audit.rs`

### Tamper-Evident Audit Trail

Merkle hash chain structure:

```
entry_hash = SHA-256(prev_hash || canonical_json)
```

- **Genesis seed**: `0000000000000000000000000000000000000000000000000000000000000000`
- **Event types**: CommandExecution, FileAccess, ConfigChange, AuthSuccess, AuthFailure, PolicyViolation, SecurityEvent
- **Optional HMAC-SHA256 signing**: Requires `ZEROCLAW_AUDIT_SIGNING_KEY` env var (32 bytes hex)
- **Log rotation**: Configurable max size with numbered backups
- **Recovery**: Reads last entry on startup to continue chain
- **Verification**: `verify_chain()` checks sequence continuity, hash linkage, and signatures

---

## 9. Observability / Security Monitoring

**Location:** `src/observability/mod.rs`

### Pluggable Observability Backend Factory

| Backend | Feature Flag | Purpose |
|---------|-------------|---------|
| `noop` | (default) | No-op observer |
| `log` | (built-in) | Structured logging via `tracing` |
| `verbose` | (built-in) | Extended logging |
| `prometheus` | `observability-prometheus` | Prometheus metrics |
| `otel` | `observability-otel` | OpenTelemetry tracing + metrics |

Factory pattern with graceful fallback when features disabled.

---

## Performance Patterns Related to Security

### Memory Budgets and Limits

| Limit | Value | Purpose |
|-------|-------|---------|
| `MAX_TRACKED_CLIENTS` | 10,000 | Pairing guard failed attempts map |
| `FAILED_ATTEMPT_RETENTION_SECS` | 900 (15 min) | Failed attempt tracking |
| `MAX_PAIR_ATTEMPTS` | 5 | Lockout threshold |
| Response cache (hot) | 256 entries | Configurable LRU |
| Response cache (warm) | Unlimited | Configurable SQLite |

### Multi-Stage Memory Retrieval

```
cache (LRU, 256 entries, 5min TTL)
    ↓ miss
fts (FTS5 keyword search)
    ↓ if top_score >= 0.85 (early return)
vector (vector similarity + hybrid merge)
```

FTS early return optimization reduces unnecessary processing.

### Response Cache (Two-Tier)

Avoids burning tokens on repeated prompts:
- **Key**: `SHA-256(model || system_prompt || user_prompt)`
- **Hot**: In-memory LRU with TTL
- **Warm**: SQLite WAL mode, indexed by `accessed_at`

---

## Key Files Reference

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
