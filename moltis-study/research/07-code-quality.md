# Moltis Code Quality Analysis

## Overview

Moltis is a 52-crate Rust monorepo with a well-developed code quality infrastructure. The project enforces rigorous standards through CI/CD gates, comprehensive testing at multiple levels, and strong conventions around error handling, secrets management, and observability.

---

## Linting and Formatting

### Clippy Configuration (`clippy.toml`)

```toml
avoid-breaking-exported-api  = false
too-many-arguments-threshold = 999
type-complexity-threshold    = 999
```

**Observations:**
- Thresholds are set very high (999), effectively disabling these lint checks at the workspace level
- `avoid-breaking-exported-api = false` is unusual; typically projects set this to `true` to prevent breaking changes to public APIs
- The project relies on targeted linting rather than global thresholds
- CI uses `cargo +nightly-2025-11-30 clippy -Z unstable-options --workspace --all-features --all-targets -- -D warnings` which **denies all warnings**

### Rustfmt Configuration (`rustfmt.toml`)

```toml
condense_wildcard_suffixes     = true
format_macro_bodies            = false
format_macro_matchers          = false
group_imports                  = "Preserve"
imports_granularity            = "One"
match_block_trailing_comma     = true
normalize_comments             = true
normalize_doc_attributes       = true
overflow_delimited_expr        = true
reorder_impl_items             = true
single_line_if_else_max_width  = 0
single_line_let_else_max_width = 0
style_edition                  = "2024"
use_field_init_shorthand       = true
use_try_shorthand              = true
```

**Observations:**
- Uses Rust 2024 style edition formatting
- Disables single-line if-else and let-else formatting (forces multiline)
- Imports granularity set to "One" (one import per use statement)
- `group_imports = "Preserve"` maintains existing import group structure
- Try shorthand (`?` operator) enabled

### CI Enforcement

The `justfile` defines:
```bash
# Format check (must match CI)
format-check:
    cargo +nightly-2025-11-30 fmt --all -- --check

# Lint (OS-aware: macOS excludes CUDA features)
lint:
    cargo +nightly-2025-11-30 clippy -Z unstable-options --workspace --all-features --all-targets -- -D warnings
```

**Key insight:** All formatting uses pinned nightly (`nightly-2025-11-30`), never stable rustfmt.

---

## Testing Infrastructure

### Test Organization

**Three testing layers:**

1. **Unit tests** — Inline in source files with `#[test]` and `#[cfg(test)]` modules
2. **Integration tests** — Files in `tests/` directories per crate
3. **E2E tests** — Playwright tests in `crates/web/ui/e2e/specs/`

### Rust Test Examples

**Inline unit tests** are prevalent throughout the codebase, particularly for:
- Serialization/deserialization (`#[test] fn test_serde_*`)
- Error message parsing
- Configuration parsing
- Mock implementations

**Example from `crates/tools/src/sandbox/tests.rs` (2,442 lines):**

```rust
#![allow(clippy::unwrap_used, clippy::expect_used)]

#[test]
fn test_normalize_cgroup_container_ref() {
    assert_eq!(
        normalize_cgroup_container_ref("docker-0123456789abcdef..."),
        Some("0123456789abcdef...".into())
    );
}

// Mock implementation for testing
struct TestSandbox { ... }

#[async_trait::async_trait]
impl Sandbox for TestSandbox {
    async fn exec(&self, _id: &SandboxId, _command: &str, _opts: &ExecOpts) -> Result<ExecResult> {
        Ok(ExecResult { stdout: "ok".into(), stderr: String::new(), exit_code: 0 })
    }
}
```

**Notable test patterns:**
- Test Sandboxes use `Arc<dyn Sandbox>` for dependency injection
- Platform-specific tests gated with `#[cfg(target_os = "macos")]`, `#[cfg(target_os = "linux")]`
- Conditional features: `#[cfg(feature = "wasm")]`, `#[cfg(feature = "vault")]`
- Async tests use `#[tokio::test]`

**Integration test example from `crates/httpd/tests/auth_middleware.rs` (1,815 lines):**

```rust
/// Start a test server with isolated temp directories for concurrent tests
async fn start_auth_server_impl(...) -> (SocketAddr, Arc<CredentialStore>, Arc<GatewayState>) {
    let tmp = tempfile::tempdir().unwrap();
    moltis_config::set_config_dir(tmp.path().to_path_buf());
    moltis_config::set_data_dir(tmp.path().to_path_buf());
    std::mem::forget(tmp); // Leak so it outlives the test
    // ... server setup
}
```

### E2E Testing (Playwright)

**Structure:**
```
crates/web/ui/e2e/
  base-test.js      # Shared Playwright fixture with error-context capture
  helpers.js        # Shared utilities
  specs/
    auth.spec.js
    chat-input.spec.js
    oauth.spec.js
    onboarding.spec.js
    providers.spec.js
    ... (20+ spec files)
  mock-oauth-server.js
```

**Base test fixture** (`base-test.js`) provides automatic error context capture:
```javascript
var test = base.extend({
    page: async ({ page, context }, use, testInfo) => {
        await use(page);
        if (testInfo.status !== testInfo.expectedStatus) {
            // Attach markdown snapshot of all open pages for CI diagnostics
            await testInfo.attach("error-context", {
                body: Buffer.from(md, "utf-8"),
                contentType: "text/markdown",
            });
        }
    },
});
```

**Helper functions** (`helpers.js`):
- `navigateAndWait(page, path)` — Navigate and wait for SPA content mount
- `watchPageErrors(page)` — Collect uncaught page errors
- `waitForWsConnected(page)` — Wait for WebSocket connection
- `createSession(page)` — Create new chat session

### Test Execution

**From `justfile`:**
```bash
# OS-aware test runner using cargo nextest
test:
    if [ "$(uname -s)" = "Darwin" ]; then
        cargo +nightly-2025-11-30 nextest run --workspace --all-features --exclude moltis-providers --exclude moltis-gateway
        cargo +nightly-2025-11-30 nextest run -p moltis-providers --features local-llm-metal
    else
        cargo +nightly-2025-11-30 nextest run --workspace --all-features
    fi

# Build-test runs Rust tests + E2E tests in parallel
build-test: build-css
    cargo +nightly-2025-11-30 build --workspace --all-features --all-targets
    cargo +nightly-2025-11-30 nextest run --all-features &  # Rust tests in background
    (cd crates/web/ui && npm run e2e) &                       # E2E tests in background
```

### Inline Test Conventions

**Existence confirmed in:**
- `crates/tools/src/cron_tool.rs`
- `crates/web/src/api.rs`
- `crates/tls/src/lib.rs`
- `crates/skills/src/registry.rs`
- `crates/skills/src/requirements.rs`
- `crates/skills/src/types.rs`
- `crates/skills/src/parse.rs`
- `crates/skills/src/prompt_gen.rs`
- `crates/skills/src/formats.rs`
- `crates/providers/src/openai_compat.rs`
- `crates/providers/src/openai.rs`
- `crates/sessions/src/store.rs`
- `crates/skills/src/discover.rs`
- `crates/providers/src/lib.rs`
- `crates/openclaw-import/src/lib.rs`
- `crates/providers/src/anthropic.rs`
- `crates/node-host/src/service.rs`
- `crates/httpd/src/server.rs`
- `crates/httpd/src/upload_routes.rs`

---

## Error Handling Patterns

### Error Type Architecture

**Central error type in `crates/common/src/error.rs`:**

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum Error {
    #[error("{0}")]
    Message(String),

    #[error(transparent)]
    Io(#[from] std::io::Error),

    #[error("internal error")]
    Other {
        #[source]
        source: Box<dyn std::error::Error + Send + Sync>,
    },
}

impl Error {
    pub fn message(message: impl Into<String>) -> Self { ... }
    pub fn other(source: impl std::error::Error + Send + Sync + 'static) -> Self { ... }
}

pub type MoltisError = Error;
pub type Result<T> = std::result::Result<T, Error>;
```

**Context trait for error enrichment:**
```rust
pub trait FromMessage: Sized {
    fn from_message(message: String) -> Self;
}

#[macro_export]
macro_rules! impl_context {
    () => {
        pub trait Context<T> {
            fn context(self, context: impl Into<String>) -> Result<T>;
            fn with_context<C, F>(self, f: F) -> Result<T> where F: FnOnce() -> C;
        }
        // ... impl for Result<T, E> and Option<T>
    };
}
```

**Usage pattern:**
```rust
moltis_common::impl_context!(); // In each crate's error module

// Then:
file_contents.context("reading config file")?;
option.ok_or_else(|| ...).with_context(|| "missing field")?;
```

### Per-Crate Error Types

Many crates define their own error types using `thiserror`:

- `crates/telegram/src/error.rs`
- `crates/tailscale/src/error.rs`
- `crates/sessions/src/error.rs`
- `crates/routing/src/error.rs`
- `crates/tls/src/error.rs`
- `crates/vault/src/error.rs`
- `crates/tools/src/error.rs`
- `crates/whatsapp/src/error.rs`
- `crates/web/src/error.rs`

### Error Handling Guidelines (from CLAUDE.md)

**Forbidden patterns:**
```rust
// FORBIDDEN: Never .unwrap()/.expect() in production
// Workspace lints deny these

// Allowed alternatives:
result?                              // Propagate
ok_or_else(|| error)                 // Option to Result
unwrap_or_default()                   // With sensible default
unwrap_or_else(|e| e.into_inner())    // For Mutex locks
```

### Logging vs Errors

**From CLAUDE.md:**
- `error!` — unrecoverable errors
- `warn!` — unexpected but recoverable
- `info!` — operational milestones
- `debug!` — detailed diagnostics
- `trace!` — very verbose per-item data

**Important note:**
> Common mistake: `warn!` for unconfigured providers — use `debug!` for expected "not configured" states.

---

## Observability

### Tracing

**Configuration:**
- `tracing` crate for structured logging
- `tracing-subscriber` with `env-filter` and `json` features
- `tracing-opentelemetry` for OTLP export
- `#[instrument]` attribute on async functions (found in `crates/chat/src/lib.rs`, `crates/tools/src/exec.rs`, `crates/gateway/src/channel.rs`)

**Best practice from CLAUDE.md:**
> All crates must have `tracing` and `metrics` features, gated with `#[cfg(feature = "...")]`. Use `tracing::instrument` on async functions.

### Metrics

**Architecture:**
- `metrics` crate facade (similar to `log` crate)
- `metrics-exporter-prometheus` for Prometheus endpoint
- `metrics-tracing-context` for span-to-metric label propagation

**Feature gates:**
```toml
[features]
metrics = ["..."]    # Enables /api/metrics JSON endpoint
prometheus = ["..."] # Enables /metrics Prometheus endpoint
```

**Prometheus endpoint:**
```
GET http://localhost:18789/metrics
```

**Available metrics categories (from `docs/src/metrics-and-tracing.md`):**
- HTTP metrics (`moltis_http_requests_total`, `moltis_http_request_duration_seconds`)
- LLM/Agent metrics (completions, tokens, errors, TTFT)
- MCP metrics (tool calls, server connections)
- Tool execution metrics
- Session metrics
- Cron job metrics
- Memory/search metrics
- Channel metrics
- OAuth metrics
- Skills metrics

**JSON API endpoints:**
- `GET /api/metrics` — Full metrics snapshot
- `GET /api/metrics/summary` — Lightweight counts
- `GET /api/metrics/history` — Time-series data (7-day retention)

---

## Secrets Management

### Secrets Handling Patterns

**`crates/common/src/secret_serde.rs`:**

```rust
use secrecy::{ExposeSecret, Secret};

/// Sentinel value for redacted secret fields in API responses
pub const REDACTED: &str = "[REDACTED]";

// Storage path (exposes raw value for persistence)
pub fn serialize_secret<S: serde::Serializer>(
    secret: &Secret<String>,
    serializer: S,
) -> Result<S::Ok, S::Error> {
    serializer.serialize_str(secret.expose_secret())
}

// Redacted path (used by per-channel RedactedConfig wrapper types)
```

**Key principles (from CLAUDE.md):**
- Use `secrecy::Secret<String>` for all passwords/keys/tokens
- `expose_secret()` only at consumption point
- Custom `Debug` impl with `[REDACTED]` to prevent accidental logging
- Scope `RwLock` read guards in blocks to avoid deadlocks

### Configuration Secrets

**`crates/config/src/schema.rs`** demonstrates proper `Secret` usage:
```rust
use secrecy::{ExposeSecret, Secret};

pub struct ProviderConfig {
    pub api_key: Option<Secret<String>>,
    // ...
}
```

---

## Configuration Management

### TOML-Based Configuration

**Config location:** `~/.moltis/moltis.toml`

**Schema validation:** `crates/config/src/schema.rs` with `crates/config/src/validate.rs`

**Key conventions:**
- When adding fields to `MoltisConfig`, also update `build_schema_map()` in validate.rs
- New enum variants need updates in `check_semantic_warnings()`
- Provider keys stored in `~/.config/moltis/provider_keys.json` via `KeyStore`

### Timezone Handling

```rust
// crates/config/src/schema.rs
use chrono_tz::Tz;

#[derive(Debug, thiserror::Error)]
#[error("unknown IANA timezone: {value}")]
pub struct TimezoneParseError { value: String }

pub struct Timezone(pub Tz);

impl std::str::FromStr for Timezone {
    type Err = TimezoneParseError;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        s.parse::<Tz>().map(Self).map_err(|_| TimezoneParseError { value: s.to_string() })
    }
}
```

---

## CI/CD Quality Gates

### GitHub Actions Workflows

1. **ci.yml** — fmt + lint + test
2. **e2e.yml** — end-to-end web UI tests
3. **release.yml** — packaging (deb, rpm, arch, appimage)
4. **docs.yml** — mdBook documentation deploy
5. **codspeed.yml** — benchmarking
6. **homebrew.yml** — Homebrew tap publish

### Pre-Merge Validation

**`./scripts/local-validate.sh <PR_NUMBER>`** must pass before PR merge.

**Exact commands enforced:**
```bash
# Rust fmt (exact command)
cargo +nightly-2025-11-30 fmt --all -- --check

# Clippy (OS-aware)
just lint

# Tests (OS-aware)
just test

# TOML formatting
taplo fmt

# JS linting/formatting
biome check --write

# Swift (macOS only)
./scripts/build-swift-bridge.sh && ./scripts/generate-swift-project.sh && ./scripts/lint-swift.sh && xcodebuild ...
```

### Conventional Commits

Enforced via `scripts/check-changelog-guard.sh`. Format:
```
feat|fix|docs|style|refactor|test|chore(scope): description
```

---

## Workspace Structure

**52 crates** organized by domain:
- **Apps:** `apps/courier` (APNS relay)
- **Core:** `agents`, `auth`, `chat`, `config`, `gateway`, `httpd`, `protocol`, `routing`
- **Channels:** `discord`, `slack`, `telegram`, `whatsapp`, `msteams`, `caldav`, `canvas`
- **Providers:** `providers` (OpenAI, Copilot, Ollama), `provider-setup`
- **Tools:** `tools` (sandbox, execution), `wasm-tools/*`
- **Services:** `cron`, `graphql`, `memory`, `oauth`, `plugins`, `sessions`, `skills`
- **Infrastructure:** `metrics`, `tls`, `vault`, `network-filter`, `browser`
- **Platform:** `swift-bridge`, `node-host`, `mcp`

**Toolchain:**
- Nightly pinned to `nightly-2025-11-30`
- Edition 2024
- Resolver v2

---

## Summary Table

| Aspect | Practice |
|--------|----------|
| **Formatting** | Nightly rustfmt, pinned toolchain, CI-enforced |
| **Linting** | Clippy with `-D warnings`, OS-aware CI |
| **Unit Tests** | Inline `#[test]`, `#[cfg(test)]`, `#[tokio::test]` |
| **Integration Tests** | Per-crate `tests/` directories |
| **E2E Tests** | Playwright with shared fixtures and helpers |
| **Error Types** | `thiserror` enums, shared `Context` trait |
| **Logging** | `tracing` with `#[instrument]`, level discipline |
| **Metrics** | Facade pattern, feature-gated, Prometheus export |
| **Secrets** | `secrecy::Secret`, redacted serde, no unwrap |
| **Config** | TOML with validation, schema-driven |
| **CI/CD** | GitHub Actions, `just` task runner, pre-merge validation |
| **Commits** | Conventional commits, changelog automation |

---

## Strengths

1. **Comprehensive test coverage** — 3 layers (unit, integration, E2E) with platform-specific and feature-gated tests
2. **Strong secrets discipline** — `secrecy` crate usage enforced, custom Debug impls, redacted serialization
3. **Observability-first** — Full tracing + metrics integration with feature gating
4. **CI-gated quality** — Every PR runs fmt, clippy, tests, and E2E checks
5. **Error context enrichment** — Shared `Context` trait with `.context()` and `.with_context()`
6. **Documentation** — `docs/src/metrics-and-tracing.md` with comprehensive metric definitions
7. **E2E error capture** — Playwright base fixture automatically attaches page context on failure

## Areas for Improvement

1. **Clippy thresholds** — Very high thresholds (999) effectively disable these checks; consider tightening or removing
2. **Test isolation** — Integration tests use `tempfile::tempdir().unwrap()` with `std::mem::forget()` which could leak if tests panic
3. **`#[allow(clippy::unwrap_used)]`** — Tests explicitly allow unwrap; while necessary for tests, consider a test-specific pattern that doesn't require attributes
4. **Documentation drift** — CLAUDE.md notes "Keep docs in sync with code" but no automated checking exists for config schema vs docs
5. **Coverage tracking** — No mention of code coverage enforcement (e.g., `cargo-llvm-cov`)
