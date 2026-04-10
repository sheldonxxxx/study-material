# Dependencies

## Dependency Map

### Async Runtime

| Dependency | Version | Purpose |
|------------|---------|---------|
| **tokio** | (full features) | Async runtime for all I/O operations |

### HTTP / Web

| Dependency | Version | Purpose |
|------------|---------|---------|
| **axum** | 0.8 | HTTP server + WebSocket |
| **axum-extra** | 0.10 | Cookie support, extensions |
| **hyper** | 1.x | Low-level HTTP primitives |
| **hyper-util** | (latest) | Hyper utilities |
| **tower** / **tower-http** | (latest) | Middleware (auth, cors, rate limit) |
| **reqwest** | 0.12 | HTTP client for LLM APIs |
| **tokio-tungstenite** | 0.26 | WebSocket client |
| **askama** | 0.15 | Server-side HTML templates |

### Serialization

| Dependency | Version | Purpose |
|------------|---------|---------|
| **serde** / **serde_json** | (latest) | JSON serialization |
| **serde_yaml** | (latest) | YAML config parsing |
| **postcard** | (alloc) | Binary serialization for WASM |
| **toml** / **toml_edit** | (latest) | TOML config handling |

### Database

| Dependency | Version | Purpose |
|------------|---------|---------|
| **sqlx** | 0.8 | Async SQLite with migrations |

### LLM / AI Providers

| Dependency | Version | Purpose |
|------------|---------|---------|
| **async-openai** | 0.32 | OpenAI API client |
| **genai** | 0.5 | Generic AI abstraction |
| **llama-cpp-2** | 0.1 | llama.cpp bindings for local models |

### Messaging Channels

| Dependency | Version | Purpose |
|------------|---------|---------|
| **slack-morphism** | 2.6 | Slack API client |
| **teloxide** | 0.13 | Telegram bot framework |
| **serenity** | 0.12 | Discord bot library |
| **wacore** / **waproto** | (latest) | WhatsApp protocol |
| **whatsapp-rust** | (latest) | WhatsApp implementation |

### GraphQL

| Dependency | Version | Purpose |
|------------|---------|---------|
| **async-graphql** | 7 | GraphQL server |
| **async-graphql-axum** | 7 | Axum integration |

### WASM / Sandboxing

| Dependency | Version | Purpose |
|------------|---------|---------|
| **wasmtime** | 36 | WASM runtime with component model |
| **wit-bindgen** | 0.41 | WIT interface bindings |
| **wasmtime-wasi** | 36 | WASI support |
| **wit-bindgen-rt** | (latest) | WIT runtime support |

### Security / Cryptography

| Dependency | Version | Purpose |
|------------|---------|---------|
| **argon2** | (latest) | Password hashing |
| **chacha20poly1305** | (latest) | Symmetric encryption |
| **p256** | 0.13 | ECDH key exchange |
| **webauthn-rs** | 0.5 | Passkey / WebAuthn |
| **secrecy** | 0.8 | Secret value wrapper |
| **rustls** | 0.23 | TLS (ring backend) |
| **openssl** | 0.10 | OpenSSL fallback (vendored) |

### Observability

| Dependency | Version | Purpose |
|------------|---------|---------|
| **tracing** / **tracing-subscriber** | (env-filter, json) | Structured logging |
| **opentelemetry** / **opentelemetry-otlp** | (latest) | OTLP export |
| **tracing-opentelemetry** | (latest) | Tracing → OTLP bridge |
| **metrics** / **metrics-exporter-prometheus** | (latest) | Prometheus metrics |

### Code Analysis / Parsing

| Dependency | Version | Purpose |
|------------|---------|---------|
| **tree-sitter** | (multiple) | Multi-language code parsing |
| **text-splitter** | 0.29 | Code chunking for embeddings |
| **gix** | 0.78 | Git operations |

### Utilities

| Dependency | Version | Purpose |
|------------|---------|---------|
| **chrono** / **chrono-tz** | (latest) | Date/time with timezones |
| **uuid** | v4 | UUID generation |
| **directories** | 6 | Platform config/data directories |
| **qrcode** | 0.14 | QR code generation |
| **image** | (jpeg, png, webp) | Image processing |
| **pulldown-cmark** | (latest) | Markdown parsing |
| **html2text** | (latest) | HTML → plain text |
| **icalendar** | 0.17 | iCalendar parsing |
| **libdav** | 0.10 | CalDAV client |
| **sled** | 0.34 | Embedded KV store |
| **tikv-jemallocator** | 0.6 | Jemalloc allocator |
| **clap** | 4 | CLI argument parsing |

## Notable Version Choices

### tokio (full features)
Enables all platform-specific features (epoll, kqueue, IOCP) for maximum async performance.

### sqlx 0.8 (sqlite)
SQLite chosen for simplicity; `sqlx` provides compile-time query validation via `cargo sqlx prepare`.

### async-graphql 7
Version 7 provides async GraphQL with subscription support; paired with `async-graphql-axum` for integration.

### wasmtime 36
Latest stable with component model support; enables WASI Preview 2 guest tools.

### rustls 0.23 (ring)
Ring backend is faster than OpenSSL; vendored OpenSSL provides fallback for systems without ring.

### serenity 0.12 (rustls)
Discord bot library with rustls for TLS; avoids OpenSSL dependency on Linux.

### tree-sitter (multiple language grammars)
Enables code chunking for memory/embeddings; supports bash, c, cpp, go, java, javascript, json, python, ruby, rust, toml, typescript.

### tikv-jemallocator 0.6
Jemalloc reduces memory fragmentation in long-running gateway process.

## Web UI Dependencies (npm)

| Dependency | Purpose |
|------------|---------|
| **Node.js 22 LTS** | JavaScript runtime |
| **Tailwind CSS v4** | Styling |
| **Playwright** | E2E testing |
| **Biome** | JS/TS linting/formatting |

## Workspace Structure Impact

With 52 crates in a Cargo workspace:
- Single `Cargo.lock` ensures compatible versions across all crates
- Resolver version 2 handles dependency resolution for complex workspaces
- Shared dependencies in root `Cargo.toml` reduce duplicate compilation
