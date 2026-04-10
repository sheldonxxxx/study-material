# OpenFang CI/CD

## Build Pipeline

### CI Workflow (`.github/workflows/ci.yml`)

Triggered on: push to `main`, all pull requests to `main`

#### Jobs

| Job | Platform | Steps |
|-----|----------|-------|
| **Check** | Ubuntu, macOS, Windows | `cargo check --workspace` |
| **Test** | Ubuntu, macOS, Windows | `cargo test --workspace` |
| **Clippy** | Ubuntu | `cargo clippy --workspace -- -D warnings` |
| **Format** | Ubuntu | `cargo fmt --check` |
| **Security Audit** | Ubuntu | `cargo audit` |
| **Secrets Scan** | Ubuntu | TruffleHog filesystem scan (with `--fail --only-verified`) |
| **Install Smoke** | Ubuntu | Shell and PowerShell installer syntax validation |

#### CI Environment

- `RUSTFLAGS: "-D warnings"` -- all warnings treated as errors
- `CARGO_TERM_COLOR: always` -- colored output
- Rust cache via `Swatinem/rust-cache@v2`
- Linux Tauri deps installed: `libwebkit2gtk-4.1-dev`, `libgtk-3-dev`, `libayatana-appindicator3-dev`, `librsvg2-dev`, `patchelf`

---

### Release Workflow (`.github/workflows/release.yml`)

Triggered on: git tags matching `v*` (e.g., `v0.5.2`)

#### Desktop App Build (Tauri 2.0)

| Platform | Artifacts | Code Signing |
|---------|-----------|--------------|
| Linux x86_64 | AppImage, .deb | None |
| macOS x86_64 | .dmg, .app | Developer ID Application |
| macOS ARM64 | .dmg, .app | Developer ID Application |
| Windows x86_64 | .msi, .exe | Windows Authenticode |
| Windows ARM64 | .msi, .exe | Windows Authenticode |

macOS signing requires:
- `MAC_CERT_BASE64` (base64-encoded .p12 certificate)
- `MAC_CERT_PASSWORD`
- Apple notarization: `APPLE_ID`, `APPLE_PASSWORD`, `APPLE_TEAM_ID`

macOS notarization is automated via `tauri-action`.

#### CLI Binary Build

| Target | OS | Archive | Notes |
|--------|-----|---------|-------|
| `x86_64-unknown-linux-gnu` | Ubuntu | tar.gz | Standard build |
| `aarch64-unknown-linux-gnu` | Ubuntu | tar.gz | Uses `cross` for ARM64 |
| `x86_64-apple-darwin` | macOS | tar.gz | Ad-hoc `codesign` |
| `aarch64-apple-darwin` | macOS | tar.gz | Ad-hoc `codesign` |
| `x86_64-pc-windows-msvc` | Windows | zip | SHA256 checksum |
| `aarch64-pc-windows-msvc` | Windows | zip | SHA256 checksum |

For ARM64 Linux: `cargo install cross --locked` then `cross build --release --target aarch64-unknown-linux-gnu --bin openfang`

#### Docker Build

- **Registry**: GHCR (`ghcr.io/rightnow-ai/openfang`)
- **Platforms**: `linux/amd64`, `linux/arm64` (multi-arch via QEMU)
- **Tags**:
  - `ghcr.io/rightnow-ai/openfang:latest`
  - `ghcr.io/rightnow-ai/openfang:<version>` (from git tag)
- **Cache**: GitHub Actions cache (`type=gha`, `mode=max`)
- **Base image**: `rust:1-slim-bookworm` (multi-stage build)

#### Release Assets Upload

- CLI binaries uploaded via `softprops/action-gh-release@v2`
- Desktop installers uploaded automatically via `tauri-action`
- SHA256 checksums generated for all artifacts

---

## Test Strategy

### Test Suite

- **Total tests**: 1,744+ (workspace-wide)
- **Command**: `cargo test --workspace`
- **Headless CI**: Tests requiring a display (Tauri) are skipped via `#[cfg]` in headless CI
- **Single-crate testing**: `cargo test -p <crate-name>`

### Test Requirements

All tests must pass before merging. CI enforces:
1. `cargo test --workspace` passes on all 3 platforms (Ubuntu, macOS, Windows)
2. `cargo clippy --workspace --all-targets -- -D warnings` produces zero warnings
3. `cargo fmt --all --check` produces no diff

---

## Platform Support

| Platform | CLI | Desktop | Docker |
|----------|-----|---------|--------|
| Linux x86_64 | Yes | AppImage, .deb | Yes |
| Linux ARM64 | Yes (cross-compiled) | N/A | Yes |
| macOS x86_64 | Yes | .dmg | N/A |
| macOS ARM64 | Yes | .dmg | N/A |
| Windows x86_64 | Yes | .msi, .exe | N/A |
| Windows ARM64 | Yes | .msi, .exe | N/A |

---

## Installation

### Shell Installer (Linux/macOS)
```bash
curl -sSf https://openfang.sh | sh
```

### PowerShell Installer (Windows)
```powershell
irm https://openfang.sh/install.ps1 | iex
```

### Docker
```bash
docker pull ghcr.io/rightnow-ai/openfang:latest
```

### From Source
```bash
git clone https://github.com/RightNow-AI/openfang.git
cd openfang
cargo build --release -p openfang-cli
```

---

## Release Cadence

The project shows **very active development** with rapid releases:

- v0.1.0 - 2026-02-24 (initial release)
- v0.5.0 - recent (feature release)
- v0.5.1 - recent (patch)
- v0.5.2 - 2026-03-26 (latest)

Approximately **5 releases in ~1 month**, indicating an active launch phase with frequent iteration.

---

## CI/CD Summary

| Aspect | Implementation |
|--------|---------------|
| **CI Platform** | GitHub Actions |
| **Test Count** | 1,744+ tests |
| **Linting** | Clippy with `-D warnings` |
| **Formatting** | rustfmt |
| **Security** | cargo-audit, TruffleHog secrets scan |
| **Desktop Build** | Tauri 2.0 with code signing + notarization |
| **CLI Targets** | 6 (Linux x64/ARM64, macOS x64/ARM64, Windows x64/ARM64) |
| **Container** | Docker multi-arch (amd64 + arm64) |
| **Installers** | Shell, PowerShell, npm |
| **Cache** | Swatinem rust-cache, Docker layer cache |
