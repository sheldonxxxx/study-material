# CI/CD

## Build System: `just`

Moltis uses [`just`](https://just.systems/) (not Make) as the task runner. The `justfile` contains ~50 recipes.

### Core Commands

| Command | Purpose |
|---------|---------|
| `just build` | Standard Rust debug build |
| `just build-release` | Optimized release build |
| `just build-css` | Compile Tailwind CSS for web UI |
| `just build-wasm-artifacts` | Build WASM guest tools + pre-compile to `.cwasm` |
| `just format` / `just format-check` | Nightly rustfmt (pinned) |
| `just lint` | Clippy with OS-aware CUDA/metal feature gates |
| `just test` | cargo nextest (parallel test runner) |
| `just ci` | Full pipeline: fmt + lint + i18n + build-css + build + test |
| `just build-test` | Parallel Rust tests + E2E tests |
| `just ui-e2e` / `just ui-e2e-headed` | Playwright E2E tests |

### Packaging Commands

| Command | Output |
|---------|--------|
| `just deb` / `just deb-amd64` / `just deb-arm64` | Debian packages |
| `just arch-pkg*` | Arch Linux packages |
| `just rpm*` | RPM packages |
| `just appimage` | AppImage |
| `just swift-*` | macOS/iOS Swift builds |
| `just ios-*` | iOS app builds |
| `just courier-*` | APNS relay builds |

## GitHub Actions Workflows

### Main CI Pipeline (`ci.yml`)

Runs on every PR and push to main:
1. **Rust fmt check** — pinned nightly rustfmt
2. **Rust clippy** — deny `expect_used`, `unwrap_used`; deny all warnings
3. **TOML formatting** — taplo validation
4. **i18n check** — translation completeness
5. **CSS build** — Tailwind CSS compilation
6. **Rust build** — full workspace compile
7. **Rust tests** — cargo nextest (parallel)

### End-to-End Tests (`e2e.yml`)

Playwright-based E2E tests for web UI:
- Runs in headed/headless modes
- Tests critical user flows (login, chat, settings)

### Release Pipeline (`release.yml`)

Very extensive (55KB) workflow that builds and publishes:
- Debian packages (.deb) via `cargo deb`
- RPM packages via `cargo generate-rpm`
- Arch Linux packages (.pkg.tar.zst)
- AppImage via `appimagetool`
- Homebrew formula updates
- Docker image pushes

### Documentation (`docs.yml`)

Deploys mdBook documentation to GitHub Pages.

### Benchmarking (`codspeed.yml`)

Codspeed integration for performance regression tracking.

### Homebrew (`homebrew.yml`)

Official Homebrew tap updates when version bumps occur.

### Security (`zizmor.yml`)

Workflow security linting via `zizmor`.

## Testing Infrastructure

### Unit Tests
- **cargo nextest**: Parallel test execution with better failure reporting
- **OS-aware test splits**: macOS splits out metal-specific features
- **Workspace isolation**: Each crate has independent tests

### Integration Tests
- **Database migrations**: sqlx migrations tested on fresh DB
- **Channel mocks**: Test each messaging channel with mock servers

### E2E Tests
- **Playwright**: Browser automation for web UI
- **Headless by default**: CI-friendly; headed mode for debugging
- **Coverage**: Critical flows (auth, chat, settings, plugins)

## Code Quality Gates

| Gate | Tool | Configuration |
|------|------|----------------|
| Rust formatting | rustfmt (nightly pinned) | `nightly-2025-11-30` |
| Rust linting | clippy | `deny` warnings, `expect_used` lints |
| TOML formatting | taplo | Enforced in CI |
| JS/TS formatting | Biome | `biome.json` config |
| Commit messages | conventional commits | Enforced via hooks |
| Pre-commit | `./scripts/local-validate.sh` | Must pass before PR |

## Release Process

### Versioning Strategy
- **Date-based**: `YYYYMMDD.NN` (e.g., `v0.10.18`)
- **Cargo.toml**: Static `0.1.0` — real version injected via `MOLTIS_VERSION` env var
- **Injection point**: Build script or CI env

### Release Cadence
- **Extremely active**: 11 releases in ~3 weeks (March 2026)
- **Typical**: Multiple releases per week during active development
- **Series**: v0.10.x (current), v0.9.0 (Feb 2026), v0.8.x (Feb 2026)

### Release Workflow
1. PR merged to main triggers version bump
2. `release.yml` dispatches on tag creation
3. Builds all platform packages (deb, rpm, arch, appimage)
4. Publishes to GitHub Releases
5. Updates Homebrew tap
6. Publishes Docker image

## Docker

### Dockerfile
- **Base**: `debian:bookworm-slim`
- **Multi-stage**: Build → runtime
- **Runtime packages**: Chromium, Node.js 22, Docker CLI, `sudo`
- **Ports**:
  - 13131: HTTPS gateway
  - 13132: HTTP (CA cert)
  - 1455: OAuth callback
- **User**: Non-root `moltis` with Docker socket access
- **Volumes**: config, data, npm cache, Docker socket

## Local Validation

Before any PR, run:
```bash
./scripts/local-validate.sh
```

This script runs the same checks as CI to ensure no wasted CI cycles.
