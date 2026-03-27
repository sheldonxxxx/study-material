# OpenFang Dependencies

## Dependency Overview

**Workspace**: 14 crates + xtask
**Lock file**: `Cargo.lock` (217,399 bytes)
**Total direct dependencies**: ~50 workspace dependencies
**Dependency manager**: Cargo

---

## Key Dependencies by Function

### Async Runtime
| Crate | Version | Purpose |
|-------|---------|---------|
| tokio | 1 (full) | Async runtime with full feature set |
| tokio-stream | 0.1 | Stream utilities for async iteration |

### HTTP Server (API Daemon)
| Crate | Version | Purpose |
|-------|---------|---------|
| axum | 0.8 (ws) | Web framework with WebSocket support |
| tower | 0.5 | Middleware foundation |
| tower-http | 0.6 | CORS, tracing, compression middleware |

### HTTP Client (LLM Drivers)
| Crate | Version | Purpose |
|-------|---------|---------|
| reqwest | 0.12 | HTTP client with JSON, streaming, multipart, rustls TLS |
| tokio-tungstenite | 0.24 | WebSocket client for Discord/Slack gateways |

### Serialization
| Crate | Version | Purpose |
|-------|---------|---------|
| serde | 1 (derive) | Serialization framework |
| serde_json | 1 | JSON serialization |
| toml | 0.8 | TOML config parsing |
| rmp-serde | 1 | MessagePack for wire protocol |
| serde_yaml | 0.9 | YAML parsing |
| json5 | 0.4 | JSON5 parsing |

### Database
| Crate | Version | Purpose |
|-------|---------|---------|
| rusqlite | 0.31 (bundled) | SQLite with bundled SQLite, JSON support |

### Security / Crypto
| Crate | Version | Purpose |
|-------|---------|---------|
| aes-gcm | 0.10 | AES-256-GCM authenticated encryption |
| argon2 | 0.5 | Password hashing |
| ed25519-dalek | 2 (rand_core) | Ed25519 digital signatures |
| hmac | 0.12 | HMAC message authentication |
| sha2 | 0.10 | SHA-256/SHA-512 hashing |
| sha1 | 0.10 | SHA-1 hashing |
| subtle | 2 | Constant-time operations |
| zeroize | 1 (derive) | Memory zeroing |

### WASM Runtime
| Crate | Version | Purpose |
|-------|---------|---------|
| wasmtime | 41 | WASM sandbox for skill isolation |

### CLI
| Crate | Version | Purpose |
|-------|---------|---------|
| clap | 4 (derive) | CLI argument parsing |
| clap_complete | 4 | Shell completion generation |
| ratatui | 0.29 | Terminal UI library |
| colored | 3 | Terminal colors |

### Email
| Crate | Version | Purpose |
|-------|---------|---------|
| lettre | 0.11 | SMTP client with tokio1, rustls-tls |
| imap | 2 | IMAP client |
| mailparse | 0.16 | MIME parsing |

### Observability
| Crate | Version | Purpose |
|-------|---------|---------|
| tracing | 0.1 | Structured logging and diagnostics |
| tracing-subscriber | 0.3 (env-filter, json) | Log subscriber with JSON/env filtering |

### Concurrency & Utilities
| Crate | Version | Purpose |
|-------|---------|---------|
| dashmap | 6 | Concurrent HashMap |
| crossbeam | 0.8 | Parallelism primitives |
| futures | 0.3 | Async abstractions |
| thiserror | 2 | Error types |
| anyhow | 1 | Error handling |
| uuid | 1 (v4, v5, serde) | ID generation |
| chrono | 0.4 (serde) | Time handling |
| chrono-tz | 0.10 | Timezone support |
| walkdir | 2 | Directory walking |
| governor | 0.8 | Rate limiting |

---

## Dependency Health Indicators

### Active Maintenance
- **tokio**: 1.x stable, actively maintained
- **axum**: 0.8.x, actively maintained by Tower maintainers
- **serde**: 1.x stable, de facto standard
- **rusqlite**: 0.31, actively maintained

### Known Security-Critical Dependencies
- **openssl**: 0.10 (vendored) -- statically compiled, no runtime libssl dependency
- **sha2**: 0.10 -- used for HMAC-SHA256 in OFP wire protocol
- **aes-gcm**: 0.10 -- used for credential vault encryption
- **argon2**: 0.5 -- password hashing

### Heavyweight Dependencies
- **wasmtime**: 41 -- adds significant compile time
- **rusqlite** (bundled): adds compile time for bundled SQLite
- **tokio** (full): increases binary size

---

## Vulnerability Posture

### Defenses
- **cargo audit**: Run in CI on every push/PR (Ubuntu job)
- **TruffleHog**: Secrets scanning on every push/PR
- **RUSTFLAGS="-D warnings"**: Warnings treated as errors in CI
- **Vendored OpenSSL**: Statically compiled, no runtime libssl dependency

### Vulnerability Management
- **dependabot**: Active on cargo, roxmltree, docker actions
- 8 automated dependabot PRs seen in commit history

### Attack Surface Considerations
- **External HTTP requests**: reqwest used for LLM API calls (user controls provider endpoints)
- **WebSocket connections**: Discord/Slack gateways (user controls credentials)
- **File system access**: Skills and agent templates loaded from disk
- **Shell execution**: Not present in core, but skills could implement it

---

## Dependency Management Practices

### Workspace Dependencies
- All shared dependencies declared in root `Cargo.toml` `[workspace.dependencies]`
- Crates reference via workspace deps, not direct version pinning
- Prevents version conflicts across crates

### Version Pinning Strategy
- Minimum Supported Rust Version (MSRV): 1.75
- Edition: 2021
- Resolver: "2"

### Lock File
- `Cargo.lock` committed to git
- Ensures reproducible builds

### Build Profiles
| Profile | LTO | Codegen | Optimization |
|---------|-----|---------|--------------|
| release | fat | 1 | 3 (stripped) |
| release-fast | thin | 8 | 2 (not stripped) |

---

## Maintenance Health

| Aspect | Status |
|--------|--------|
| Lock file up to date | Yes (217KB lock file) |
| Dependabot active | Yes (8 PRs merged) |
| Security audits | Yes (cargo audit in CI) |
| Vulnerability disclosure | Yes (SECURITY.md at root) |
| Dependency update policy | Implicit: update on dependabot PRs |

---

## Dependencies with Known Risks

| Dependency | Risk | Mitigation |
|------------|------|------------|
| wasmtime (41) | Large compile time | Pre-built in release profile |
| rusqlite (bundled) | Large compile time | Bundled avoids system SQLite dependency |
| openssl (vendored) | Build complexity | Vendored flag simplifies builds |
| tokio (full) | Binary bloat | Only enabled in final binary, not library crates |
