# CI/CD

## Overview

ZeroClaw uses **GitHub Actions** for all CI/CD. The repository has a single source of truth branch (`master`) with automated quality gates, multi-platform builds, and a comprehensive release pipeline.

## Quality Gates

### Pull Request Checks (`checks-on-pr.yml`)

Runs on every PR and push to `master`:

- **actionlint** - GitHub Actions workflow validation
- **Label policy** - Enforces consistent labeling
- **PR template** - Ensures PR description completeness

### CI Pipeline (`ci-run.yml`)

Runs on push to `master` and all PRs:

| Job | Timeout | Purpose |
|-----|---------|---------|
| `lint` | 10 min | `cargo fmt`, `cargo clippy -- -D warnings` |
| `bench-compile` | 15 min | Verify benchmarks compile |
| `lint-strict-delta` | 15 min | Strict clippy on changed lines only |
| `test` | 30 min | `cargo nextest run --locked` with mold linker |
| `build` | 40 min | Cross-platform release builds |
| `check-all-features` | 20 min | `cargo check --features ci-all` |
| `docs-quality` | 10 min | Markdown lint on changed lines |
| `gate` | - | Composite status (blocks merge) |

**Build Matrix:**
- Ubuntu x86_64 (mold linker)
- macOS ARM64 (aarch64-apple-darwin)
- Windows x86_64 (MSVC)

### Strict Delta Lint

Runs only on changed lines to reduce noise:

```bash
bash scripts/ci/rust_strict_delta_gate.sh
```

Uses `BASE_SHA` from PR base or commit range.

## Release Pipeline

### Stable Release (`release-stable-manual.yml`)

Triggered by:
- Version tag push (`v[0-9]+.[0-9]+.[0-9]+`)
- Manual workflow dispatch

**Jobs:**

1. **validate** - Verify semver and Cargo.toml version match
2. **web** - Build web dashboard (Node 22, npm)
3. **release-notes** - Auto-generate from conventional commits
4. **build** - Multi-platform binary builds:
   - Linux: x86_64, aarch64, armv7, armv6
   - macOS: aarch64
   - Windows: x86_64
   - Android: aarch64-linux-android
5. **build-desktop** - macOS Universal binary via Tauri
6. **publish** - Create GitHub release with checksums
7. **crates-io** - Publish to crates.io (aardvark-sys, zeroclaw)
8. **docker** - Multi-arch Docker images (amd64, arm64)
9. **redeploy-website** - Trigger zeroclaw-website rebuild
10. **scoop** - Update Scoop Windows package manager
11. **aur** - Update Arch User Repository
12. **homebrew** - Update Homebrew Core
13. **tweet** - Announce on Twitter
14. **discord** - Announce on Discord

### Beta Release (`release-beta-on-push.yml`)

Triggered by version tags with `-beta` suffix (e.g., `v0.6.0-beta.1`).

### Discord Release (`discord-release.yml`)

Posts release announcement to Discord webhook.

### Homebrew/Scoop/AUR Updates

Separate reusable workflows that create/update package manifests.

## Dependency Management

### Dependabot (`dependabot.yml`)

Daily dependency updates with grouping:

| Ecosystem | Schedule | Limit | Labels |
|-----------|----------|-------|--------|
| Cargo | Daily | 3 PRs | `dependencies` |
| GitHub Actions | Daily | 1 PR | `ci`, `dependencies` |
| Docker | Daily | 1 PR | `ci`, `dependencies` |

## Pre-push Hook

Enforce quality locally before push:

```bash
git config core.hooksPath .githooks
```

Runs: `cargo fmt`, `cargo clippy`, `cargo test`

Optional strict passes:
```bash
ZEROCLAW_STRICT_LINT=1 git push        # Full strict clippy
ZEROCLAW_STRICT_DELTA_LINT=1 git push  # Changed lines only
ZEROCLAW_DOCS_LINT=1 git push          # Markdown quality gate
ZEROCLAW_DOCS_LINKS=1 git push         # Link validation
```

## Binary Size Checks

`scripts/ci/check_binary_size.sh` validates release binaries against size limits per target.

## Docker

- `Dockerfile` - Default image
- `Dockerfile.ci` - CI build environment
- `Dockerfile.debian` - Debian-compatible variant
- `docker-compose.yml` - Local development setup

## Build Artifacts

| Artifact | Retention | Platform |
|----------|-----------|---------|
| `zeroclaw-*.tar.gz` | 14 days | Linux/macOS |
| `zeroclaw-*.zip` | 14 days | Windows |
| `ZeroClaw.dmg` | 14 days | macOS Desktop |
| `web-dist` | 1 day | Build artifact |

## Concurrency

PR runs cancel in-progress on new push. Release runs do not cancel (prevents race conditions).

```yaml
concurrency:
  group: ci-${{ github.event.pull_request.number || github.sha }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```

## CodeQL

Security analysis via `.github/codeql/` configuration.
