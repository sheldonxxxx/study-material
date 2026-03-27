# ZeroClaw Tech Stack Analysis

## Project Overview

**ZeroClaw** is a Rust-based AI agent/assistant CLI tool. The repository is located at `/Users/sheldon/Documents/claw/reference/zeroclaw/`.

Key description from Cargo.toml:
> "Zero overhead. Zero compromise. 100% Rust. The fastest, smallest AI assistant."

## 1. Rust Version & Edition

| Property | Value |
|----------|-------|
| Minimum Rust Version | 1.87 (`rust-version` field) |
| CI/Release Rust Version | 1.94 (Dockerfile.builder) / 1.92.0 (GitHub Actions) |
| Edition | 2021 |

**Note:** The project uses a newer Rust version in CI (1.92.0) than the declared minimum (1.87). The Dockerfile uses rust:1.94-slim.

## 2. Workspace Structure

```
zeroclaw/
├── Cargo.toml           # Root workspace manifest
├── Cargo.lock
├── build.rs             # Build script (web frontend integration)
├── src/                 # Main application source
├── crates/
│   ├── aardvark-sys/    # Low-level bindings for Total Phase Aardvark I2C/SPI/GPIO USB adapter
│   └── robot-kit/        # Robot control toolkit (drive, vision, speech, sensors, safety)
├── apps/
│   └── tauri/           # Tauri desktop app wrapper
├── fuzz/                # Fuzzing targets
├── benches/             # Benchmark suite
├── web/                 # Frontend (React + Vite)
├── tests/               # Integration and system tests
└── .github/workflows/   # CI/CD pipelines
```

**Workspace Members:**
```toml
[workspace]
members = [".", "crates/robot-kit", "crates/aardvark-sys", "apps/tauri"]
resolver = "2"
```

## 3. Key Rust Dependencies

### Core Application

| Dependency | Version | Purpose |
|------------|---------|---------|
| `clap` | 4.5 | CLI argument parsing with derive macros |
| `tokio` | 1.50 | Async runtime (multi-threaded, with macros, time, net, io, fs, signal) |
| `reqwest` | 0.12 | HTTP client (rustls-tls, blocking, multipart, streaming, socks support) |
| `serde` / `serde_json` | 1.0 | Serialization framework |
| `directories` | 6.0 | Platform-specific directory paths |
| `toml` | 1.0 | TOML config file parsing |

### HTTP Server (Gateway)

| Dependency | Version | Purpose |
|------------|---------|---------|
| `axum` | 0.8 | Web framework (http1, json, tokio, query, websockets, macros) |
| `hyper` | 1 | HTTP library (http1, server) |
| `hyper-util` | 0.1 | HTTP utilities (tokio, server-auto, graceful shutdown) |
| `tower` / `tower-http` | 0.5 / 0.6 | Middleware (rate limiting, timeout) |
| `http-body-util` | 0.1 | HTTP body utilities |

### Data & Persistence

| Dependency | Version | Purpose |
|------------|---------|---------|
| `rusqlite` | 0.37 | SQLite bindings (bundled) |
| `chrono` / `chrono-tz` | 0.4 / 0.10 | Date/time with timezone support |
| `uuid` | 1.22 | UUID generation (v4) |

### Security

| Dependency | Version | Purpose |
|------------|---------|---------|
| `chacha20poly1305` | 0.10 | AEAD encryption for secret store |
| `hmac` / `sha2` | 0.12 / 0.10 | HMAC-SHA256 for webhook signature verification |
| `ring` | 0.17 | HMAC-SHA256 for Zhipu/GLM JWT auth |
| `rustls` | 0.23 | TLS implementation (rustls-pki-types, tokio-rustls) |

### Communication Channels (Optional Features)

| Dependency | Version | Purpose |
|------------|---------|---------|
| `matrix-sdk` | 0.16 | Matrix client with E2EE, rustls-tls, sqlite |
| `nostr-sdk` | 0.44 | Nostr protocol (nip04, nip59) |
| `tokio-tungstenite` | 0.29 | WebSocket client (rustls-tls-webpki-roots) |
| `wa-rs` | 0.2 | WhatsApp Web client (optional feature `whatsapp-web`) |
| `lettre` | 0.11 | Email (SMTP, rustls-tls) |
| `async-imap` | 0.11 | IMAP client |

### Hardware & Peripherals

| Dependency | Version | Purpose |
|------------|---------|---------|
| `tokio-serial` | 5 | Serial port communication (STM32, etc.) |
| `nusb` | 0.2 | USB device enumeration (Linux, macOS, Windows) |
| `probe-rs` | 0.31 | STM32/Nucleo memory read (optional) |
| `rppal` | 0.22 | Raspberry Pi GPIO (Linux only, optional) |
| `landlock` | 0.4 | Linux sandboxing (optional) |
| `aardvark-sys` | 0.1 | Total Phase Aardvark adapter bindings |

### Observability

| Dependency | Version | Purpose |
|------------|---------|---------|
| `tracing` / `tracing-subscriber` | 0.1 / 0.3 | Logging (fmt, ansi, env-filter) |
| `prometheus` | 0.14 | Metrics (optional feature `observability-prometheus`) |
| `opentelemetry` | 0.31 | OpenTelemetry tracing + metrics (optional feature `observability-otel`) |

### Other Notable Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| `rust-embed` | 8 | Embed frontend assets into binary |
| `image` | 0.25 | Image processing (jpeg, png) |
| `indicatif` | 0.18 | Progress bars |
| `dialoguer` | 0.12 | Interactive CLI prompts |
| `qrcode` | 0.14 | Terminal QR rendering (WhatsApp pairing) |
| `extism` | 1.20 | WASM plugin runtime (optional) |
| `cpal` | 0.15 | Audio capture for voice wake word (optional) |
| `pdf-extract` | 0.10 | PDF extraction for datasheet RAG (optional) |

## 4. Feature Flags

### Default Features
```toml
default = ["observability-prometheus", "skill-creation"]
```

### Optional Feature Groups

| Feature | Enables |
|---------|---------|
| `channel-matrix` | Matrix messaging |
| `channel-lark` / `channel-feishu` | Lark/Feishu integration |
| `channel-nostr` | Nostr protocol |
| `hardware` | USB + serial support |
| `peripheral-rpi` | Raspberry Pi GPIO |
| `browser-native` | Native browser automation (fantoccini) |
| `sandbox-landlock` / `sandbox-bubblewrap` | Linux sandboxing |
| `probe` | STM32 memory read |
| `rag-pdf` | PDF ingestion |
| `skill-creation` | Autonomous skill creation |
| `whatsapp-web` | WhatsApp Web client |
| `voice-wake` | Voice wake word detection |
| `plugins-wasm` | WASM plugin system |
| `webauthn` | FIDO2 hardware key auth |
| `ci-all` | All features except those requiring system C libraries |

## 5. Build Profiles

### Release Profile (Size-Optimized)
```toml
[profile.release]
opt-level = "z"      # Optimize for size
lto = "fat"          # Maximum cross-crate optimization
codegen-units = 1    # Serialized codegen for low-memory devices
strip = true         # Remove debug symbols
panic = "abort"      # Reduce binary size
```

### Release-Fast Profile
```toml
[profile.release-fast]
inherits = "release"
codegen-units = 8    # Parallel codegen for faster builds on powerful machines
```

### CI Profile
```toml
[profile.ci]
inherits = "release"
lto = "thin"         # Faster than fat LTO, catches release-mode issues
codegen-units = 16   # Full parallelism for CI runners
```

## 6. Build.rs Analysis

The `build.rs` script handles **web frontend integration**:

1. **Watched Files:** Triggers rebuild when web source files change:
   - `web/src`, `web/public`, `web/index.html`
   - `web/package.json`, `web/package-lock.json`
   - `web/tsconfig*.json`, `web/vite.config.ts`
   - `docs/assets/zeroclaw-trans.png`

2. **Build Logic:**
   - Checks if `web/dist` exists and is up-to-date
   - If `npm` is available and build is needed, runs `npm ci` then `npm run build`
   - Falls back gracefully if Node.js is not installed (CI containers, cross-compilation)
   - Ensures `web/dist` directory exists for `rust-embed`
   - Copies `zeroclaw-trans.png` to `web/dist/`

## 7. Web Frontend Stack

**Location:** `/web/`

### Package.json
```json
{
  "name": "zeroclaw-web",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "build": "tsc -b && vite build"
  },
  "dependencies": {
    "lucide-react": "^0.468.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "react-markdown": "^10.1.0",
    "react-router-dom": "^7.1.1",
    "remark-gfm": "^4.0.1"
  },
  "devDependencies": {
    "@tailwindcss/vite": "^4.0.0",
    "@types/node": "^25.3.0",
    "@types/react": "^19.0.7",
    "@types/react-dom": "^19.0.3",
    "@vitejs/plugin-react": "^4.3.4",
    "rollup": "^4.59.0",
    "tailwindcss": "^4.0.0",
    "typescript": "~5.7.2",
    "vite": "^6.0.7"
  }
}
```

### Frontend Technology Summary

| Category | Technology | Version |
|----------|------------|---------|
| Framework | React | 19.0.0 |
| Bundler | Vite | 6.0.7 |
| Language | TypeScript | 5.7.2 |
| CSS | Tailwind CSS | 4.0.0 |
| Icons | Lucide React | 0.468.0 |
| Markdown | react-markdown | 10.1.0 |
| Routing | React Router | 7.1.1 |

## 8. Docker Strategy

### Multi-Stage Build Architecture

**Dockerfile (Default - Distroless Production)**

| Stage | Base Image | Purpose |
|-------|------------|---------|
| web-builder | `node:22-alpine` | Build React frontend |
| builder | `rust:1.94-slim` | Compile Rust application |
| dev | `debian:trixie-slim` | Development runtime (with shell) |
| release | `gcr.io/distroless/cc-debian13:nonroot` | Production runtime (minimal) |

**Dockerfile.debian (Shell-Equipped Variant)**
- Uses `debian:bookworm-slim` as runtime
- Includes: bash, ca-certificates, curl, git
- Suitable for full coding assistant operations

### Runtime Configuration
- Default gateway port: **42617**
- Workspace directory: `/zeroclaw-data/workspace`
- Config directory: `/zeroclaw-data/.zeroclaw/config.toml`
- Default user: UID 65534 (nonroot)
- Healthcheck: `zeroclaw status --format=exit-code`

### Docker Compose
- Uses `ghcr.io/zeroclaw-labs/zeroclaw:latest` image
- Resource limits: 2 CPU, 512MB memory (limits); 0.5 CPU, 32MB (reservations)
- Environment-based configuration (API_KEY, PROVIDER, MODEL, etc.)

## 9. CI/CD Pipeline

**Location:** `.github/workflows/`

### CI Jobs

| Job | Timeout | Purpose |
|-----|---------|---------|
| `lint` | 10 min | Format check + clippy |
| `bench-compile` | 15 min | Verify benchmarks compile |
| `lint-strict-delta` | 15 min | Strict delta lint gate |
| `test` | 30 min | Run tests with cargo nextest + mold linker |
| `build` | 40 min | Cross-platform release builds (Linux x86_64, macOS ARM64, Windows x86_64) |
| `check-all-features` | 20 min | Full feature combination check |
| `docs-quality` | 10 min | Documentation quality gate |
| `gate` | - | Composite status check for branch protection |

### CI Tooling
- **Rust Toolchain:** 1.92.0 (via dtolnay/rust-toolchain)
- **Linker:** mold (faster than system linker)
- **Test Runner:** cargo-nextest
- **Caching:** Swatinem/rust-cache@v2

### Additional CI Workflows
- `checks-on-pr.yml` - PR validation
- `release-stable-manual.yml` - Manual stable releases
- `release-beta-on-push.yml` - Beta releases
- `publish-crates.yml` - Crates.io publishing
- `discord-release.yml` - Discord notifications
- `pub-homebrew-core.yml` - Homebrew tap updates
- `pub-scoop.yml` - Scoop bucket updates
- `pub-aur.yml` - AUR package updates
- `cross-platform-build-manual.yml` - Manual cross-platform builds
- `pr-path-labeler.yml` - Auto-label PRs by path

## 10. Nix Integration

**Location:** `flake.nix`

Uses **fenix** for Rust toolchain management:
```nix
fenix = {
  url = "github:nix-community/fenix";
  inputs.nixpkgs.follows = "nixpkgs";
};
```

Provides:
- `packages.default` - Fenix stable toolchain
- `devShells.default` - Development shell with rust-analyzer
- NixOS configurations for both x86_64-linux and aarch64-linux

## 11. Project Metadata

| Property | Value |
|----------|-------|
| Package Name | `zeroclawlabs` |
| Version | 0.6.3 |
| License | MIT OR Apache-2.0 |
| Authors | theonlyhennygod |
| Repository | https://github.com/zeroclaw-labs/zeroclaw |
| Categories | command-line-utilities, api-bindings |
| Keywords | ai, agent, cli, assistant, chatbot |

## 12. Notable Architecture Decisions

### Unsafe Code Policy
> "This crate is the ONLY place in ZeroClaw where unsafe code is permitted. The rest of the workspace remains #![forbid(unsafe_code)]."

Only `crates/aardvark-sys` permits unsafe code (for FFI bindings to Total Phase SDK).

### Size Optimization Priority
The project prioritizes binary size heavily:
- `opt-level = "z"` instead of speed optimization
- `lto = "fat"` for maximum cross-crate optimization
- `panic = "abort"` to reduce binary size
- Serialized codegen (`codegen-units = 1`) for low-memory device support

### Rustls Over OpenSSL
All TLS uses `rustls` instead of OpenSSL, avoiding C library dependencies.

### Bundled SQLite
Uses `rusqlite` with `features = ["bundled"]` for self-contained database support.

### Embedded Frontend
The React web dashboard is embedded into the Rust binary at compile time via `rust-embed`, served under `/_app/*` routes.

### Feature-Driven Dependency Management
Heavy use of optional dependencies gated behind features to minimize default binary size.

## 13. Development Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| `tempfile` | 3.26 | Temporary files for tests |
| `criterion` | 0.8 | Benchmarking (async_tokio) |
| `wiremock` | 0.6 | HTTP mocking |
| `scopeguard` | 1.2 | Scope protection |
| `rcgen` | 0.13 | Certificate generation |

## 14. Test Configuration

| Test Suite | Path |
|------------|------|
| Component Tests | `tests/test_component.rs` |
| Integration Tests | `tests/test_integration.rs` |
| System Tests | `tests/test_system.rs` |
| Live Tests | `tests/test_live.rs` |
| Benchmarks | `benches/agent_benchmarks.rs` |
