# Code Quality Practices

## Overview

Moltis maintains rigorous quality standards through CI/CD gates, comprehensive testing, and strong conventions around error handling and observability.

---

## Tooling

### Rust Tooling

| Tool | Configuration | Purpose |
|------|---------------|---------|
| **rustfmt** | `rustfmt.toml` (nightly, pinned) | Code formatting |
| **Clippy** | `clippy.toml` | Linting |
| **TOML fmt** | taplo | TOML file formatting |
| **Biome** | `biome.json` | JS/TS linting |

### Key Conventions

- **Pinned nightly** to `nightly-2025-11-30` for all formatting/linting
- **CI denies all warnings** via `cargo clippy -- -D warnings`
- **Conventional commits** enforced via pre-commit hooks

### Clippy Configuration

```toml
avoid-breaking-exported-api  = false
too-many-arguments-threshold = 999
type-complexity-threshold    = 999
```

Thresholds are set high - the project relies on targeted linting rather than global thresholds.

---

## Testing

### Three Testing Layers

| Layer | Location | Framework |
|-------|----------|-----------|
| **Unit** | Inline in source files | `#[test]`, `#[tokio::test]` |
| **Integration** | `tests/` directories per crate | cargo nextest |
| **E2E** | `crates/web/ui/e2e/specs/` | Playwright |

### Test Execution

```bash
# Rust tests (OS-aware)
just test

# E2E tests
cd crates/web/ui && npm run e2e

# Build + test combined
just build-test
```

### Unit Test Patterns

```rust
#[test]
fn test_normalize_cgroup_container_ref() {
    assert_eq!(
        normalize_cgroup_container_ref("docker-0123456789abcdef..."),
        Some("0123456789abcdef...".into())
    );
}

// Platform-specific tests
#[cfg(target_os = "macos")]
#[test]
fn test_macos_feature() { ... }

// Feature-gated tests
#[cfg(feature = "wasm")]
#[test]
fn test_wasm_feature() { ... }
```

### Integration Test Patterns

```rust
async fn start_auth_server_impl(...) -> (SocketAddr, Arc<CredentialStore>, Arc<GatewayState>) {
    let tmp = tempfile::tempdir().unwrap();
    moltis_config::set_config_dir(tmp.path().to_path_buf());
    std::mem::forget(tmp); // Leak so it outlives the test
    // ... server setup
}
```

### E2E Testing (Playwright)

**Base fixture** provides automatic error context capture:
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

**Helper functions:**
- `navigateAndWait(page, path)` - Navigate and wait for SPA content mount
- `watchPageErrors(page)` - Collect uncaught page errors
- `waitForWsConnected(page)` - Wait for WebSocket connection
- `createSession(page)` - Create new chat session

---

## Error Handling

### Error Type Architecture

**Central error type** in `crates/common/src/error.rs`:

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

pub type Result<T> = std::result::Result<T, Error>;
```

### Context Trait for Error Enrichment

```rust
pub trait FromMessage: Sized {
    fn from_message(message: String) -> Self;
}

// Usage:
file_contents.context("reading config file")?;
option.ok_or_else(|| ...).with_context(|| "missing field")?;
```

### Per-Crate Error Types

Many crates define their own error types using `thiserror`:
- `crates/telegram/src/error.rs`
- `crates/tailscale/src/error.rs`
- `crates/sessions/src/error.rs`
- `crates/vault/src/error.rs`
- `crates/web/src/error.rs`

### Forbidden Patterns

```rust
// FORBIDDEN: Never .unwrap()/.expect() in production
// Workspace lints deny these

// Allowed alternatives:
result?                              // Propagate
ok_or_else(|| error)                 // Option to Result
unwrap_or_default()                  // With sensible default
unwrap_or_else(|e| e.into_inner())   // For Mutex locks
```

---

## Observability

### Tracing

- `tracing` crate for structured logging
- `#[instrument]` attribute on async functions
- `tracing-opentelemetry` for OTLP export

```rust
// Best practice from CLAUDE.md:
#[instrument]
async fn exec(&self, id: &SandboxId, command: &str) -> Result<ExecResult> {
    // ...
}
```

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

### Logging Levels

| Level | Use |
|-------|-----|
| `error!` | Unrecoverable errors |
| `warn!` | Unexpected but recoverable |
| `info!` | Operational milestones |
| `debug!` | Detailed diagnostics |
| `trace!` | Very verbose per-item data |

**Important:** Use `debug!` for expected "not configured" states, not `warn!`.

---

## Secrets Management

### Using `secrecy` Crate

```rust
use secrecy::{ExposeSecret, Secret};

pub const REDACTED: &str = "[REDACTED]";

// Storage path (exposes raw value for persistence)
pub fn serialize_secret<S: serde::Serializer>(
    secret: &Secret<String>,
    serializer: S,
) -> Result<S::Ok, S::Error> {
    serializer.serialize_str(secret.expose_secret())
}
```

### Key Principles

- Use `Secret<String>` for all passwords, keys, tokens
- `expose_secret()` only at consumption point
- Custom `Debug` impl with `[REDACTED]` to prevent accidental logging
- Scope `RwLock` read guards in blocks to avoid deadlocks

---

## Configuration Management

### TOML-Based Configuration

**Config location:** `~/.moltis/moltis.toml`

**Key conventions:**
- When adding fields to `MoltisConfig`, also update `build_schema_map()` in validate.rs
- New enum variants need updates in `check_semantic_warnings()`
- Provider keys stored in `~/.config/moltis/provider_keys.json` via `KeyStore`

### Timezone Handling

```rust
use chrono_tz::Tz;

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

1. **ci.yml** - fmt + lint + test
2. **e2e.yml** - end-to-end web UI tests
3. **release.yml** - packaging (deb, rpm, arch, appimage)
4. **docs.yml** - mdBook documentation deploy
5. **codspeed.yml** - benchmarking
6. **homebrew.yml** - Homebrew tap publish

### Pre-Merge Validation

**`./scripts/local-validate.sh <PR_NUMBER>`** must pass before PR merge.

---

## Summary

| Aspect | Practice |
|--------|----------|
| Formatting | Nightly rustfmt, pinned toolchain, CI-enforced |
| Linting | Clippy with `-D warnings`, OS-aware CI |
| Unit Tests | Inline `#[test]`, `#[cfg(test)]`, `#[tokio::test]` |
| Integration Tests | Per-crate `tests/` directories |
| E2E Tests | Playwright with shared fixtures and helpers |
| Error Types | `thiserror` enums, shared `Context` trait |
| Logging | `tracing` with `#[instrument]`, level discipline |
| Metrics | Facade pattern, feature-gated, Prometheus export |
| Secrets | `secrecy::Secret`, redacted serde, no unwrap |
| CI/CD | GitHub Actions, `just` task runner, pre-merge validation |
