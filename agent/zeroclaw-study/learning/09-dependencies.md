# Dependencies

## Dependency Philosophy

ZeroClaw uses **dependency minimization** as a core principle:
- Default features disabled where possible
- Heavy use of `default-features = false`
- Bundled SQLite to avoid system library conflicts
- `rustls` instead of OpenSSL for TLS
- Size-optimized release profile (`opt-level = "z"`, `lto = "fat"`)

## Core Dependencies

### Async Runtime
```toml
tokio = { version = "1.50", default-features = false, features = ["rt-multi-thread", "macros", "time", "net", "io-util", "sync", "process", "io-std", "fs", "signal"] }
tokio-util = { version = "0.7", default-features = false }
tokio-stream = { version = "0.1.18", default-features = false, features = ["fs", "sync"] }
```

### HTTP
```toml
reqwest = { version = "0.12", default-features = false, features = ["json", "rustls-tls", "blocking", "multipart", "stream", "socks"] }
axum = { version = "0.8", default-features = false, features = ["http1", "json", "tokio", "query", "ws", "macros"] }
hyper = { version = "1", features = ["http1", "server"] }
```

### Serialization
```toml
serde = { version = "1.0", default-features = false, features = ["derive"] }
serde_json = { version = "1.0", default-features = false, features = ["std"] }
```

## Feature Flags

20+ optional features organized into groups:

| Feature | Enables | Notable Dependencies |
|---------|---------|---------------------|
| `channel-matrix` | Matrix messaging | `matrix-sdk` |
| `channel-lark` | Lark/Feishu | `prost` |
| `channel-nostr` | Nostr | `nostr-sdk` |
| `whatsapp-web` | WhatsApp client | `wa-rs`, `wa-rs-core`, `qrcode` |
| `hardware` | USB/Serial | `nusb`, `tokio-serial` |
| `peripheral-rpi` | Pi GPIO | `rppal` |
| `browser-native` | Browser automation | `fantoccini` |
| `observability-prometheus` | Metrics | `prometheus` |
| `observability-otel` | Tracing | `opentelemetry`, `opentelemetry-otlp` |
| `plugins-wasm` | WASM runtime | `extism` |
| `voice-wake` | Audio capture | `cpal` |
| `rag-pdf` | PDF ingestion | `pdf-extract` |
| `probe` | STM32 debug | `probe-rs` |
| `skill-creation` | Auto skill gen | (built-in) |

### Meta Features

```toml
ci-all = [
    "channel-nostr", "hardware", "channel-matrix", "channel-lark",
    "observability-prometheus", "observability-otel", "peripheral-rpi",
    "browser-native", "sandbox-landlock", "sandbox-bubblewrap",
    "probe", "rag-pdf", "skill-creation", "whatsapp-web", "plugins-wasm",
]
```

## Workspace Members

```toml
[workspace]
members = [".", "crates/robot-kit", "crates/aardvark-sys", "apps/tauri"]
resolver = "2"
```

### Internal Crates

| Crate | Purpose |
|-------|---------|
| `aardvark-sys` | Total Phase Aardvark USB adapter bindings |
| `robot-kit` | Robotics kit support |
| `apps/tauri` | Desktop GUI (Tauri + React) |

## Security & Licensing

### Allowed Licenses (`deny.toml`)

```
MIT, Apache-2.0, BSD-2-Clause, BSD-3-Clause, ISC, Unicode-3.0,
OpenSSL, Zlib, MPL-2.0, CDLA-Permissive-2.0, 0BSD, BSL-1.0, CC0-1.0
```

### Active Security Advisories (Ignored)

| Advisory | Reason |
|----------|--------|
| RUSTSEC-2025-0141 | bincode via probe-rs (project ceased) |
| RUSTSEC-2024-0384 | rust-nostr/nostr WIP fix |
| RUSTSEC-2024-0388 | extism wasmtime transitive |
| RUSTSEC-2025-0057 | extism wasmtime transitive |
| RUSTSEC-2025-0119 | indicatif cosmetic dep |
| RUSTSEC-2026-0006 | extism wasmtime segfault (awaiting fix) |
| RUSTSEC-2026-0020 | extism WASI resource exhaustion |
| RUSTSEC-2026-0021 | extism WASI http fields panic |

### Security Configuration

```toml
[bans]
multiple-versions = "warn"  # Warn but allow
wildcards = "allow"         # Allow wildcard deps

[sources]
unknown-registry = "deny"
unknown-git = "deny"
allow-registry = ["https://github.com/rust-lang/crates.io-index"]
allow-git = []
```

## Notable Optional Dependencies

### Database
```toml
rusqlite = { version = "0.37", features = ["bundled"] }
```

### Cryptography
```toml
chacha20poly1305 = "0.10"  # AEAD for secret store
ring = "0.17"               # JWT HMAC-SHA256
rustls = "0.23"             # TLS 1.2/1.3
```

### Messaging
```toml
matrix-sdk = { version = "0.16", optional = true, features = ["e2e-encryption", "rustls-tls", "markdown", "sqlite"] }
nostr-sdk = { version = "0.44", optional = true, features = ["nip04", "nip59"] }
wa-rs = { version = "0.2", optional = true }  # WhatsApp Web
```

### Email
```toml
lettre = { version = "0.11.19", features = ["builder", "smtp-transport", "rustls-tls"] }
async-imap = { version = "0.11", features = ["runtime-tokio"] }
```

## Development Dependencies

```toml
tempfile = "3.26"
criterion = { version = "0.8", features = ["async_tokio"] }
wiremock = "0.6"
scopeguard = "1.2"
rcgen = "0.13"
```

## Tests

Four test suites defined in `Cargo.toml`:

```toml
[[test]]
name = "component"
path = "tests/test_component.rs"

[[test]]
name = "integration"
path = "tests/test_integration.rs"

[[test]]
name = "system"
path = "tests/test_system.rs"

[[test]]
name = "live"
path = "tests/test_live.rs"

[[bench]]
name = "agent_benchmarks"
harness = false
```
