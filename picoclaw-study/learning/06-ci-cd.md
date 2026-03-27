# CI/CD

## Overview

PicoClaw uses GitHub Actions for continuous integration and delivery. Releases are automated with GoReleaser, which handles multi-platform builds, Docker images, and package managers.

## Workflows

### PR Workflow (`pr.yml`)

Runs on every pull request. Three parallel jobs:

| Job | Purpose | Key Steps |
|-----|---------|-----------|
| `lint` | Code quality | `go generate ./...`, then `golangci-lint` with `goolm,stdjson` tags |
| `vuln_check` | Security | `golang/govulncheck-action` scanning `./...` |
| `test` | Unit tests | `go generate ./...`, then `go test -tags goolm,stdjson ./...` |

All three jobs must pass before a PR can be merged.

### Release Workflow (`release.yml`)

Triggered manually via `workflow_dispatch`. Three sequential jobs:

1. **`create-tag`** — Creates and pushes a Git tag from the provided version string
2. **`release`** (needs `create-tag`) — Runs GoReleaser:
   - Builds for Linux (amd64, arm64, riscv64, loong64, armv6/7, mipsle), macOS (amd64, arm64), Windows (amd64), FreeBSD, NetBSD
   - Generates tar.gz archives (zip for Windows)
   - Builds Docker images pushed to both `ghcr.io` and `docker.io`
   - Creates `.deb` and `.rpm` packages via nfpm
   - macOS code signing and notarization (if secrets provided)
3. **`upload-tos`** (needs `release`, optional) — Uploads release artifacts to Volcengine TOS

GoReleaser configuration (`.goreleaser.yaml`):
- Three builds: `picoclaw` (main), `picoclaw-launcher` (Web UI), `picoclaw-launcher-tui` (TUI)
- Two Docker images: `picoclaw` and `picoclaw-launcher` (multi-platform: linux/amd64, linux/arm64, linux/riscv64)
- Squash merge strategy assumed for clean commit history

### Nightly Workflow (`nightly.yml`)

Runs daily at midnight UTC via cron. Produces unstable nightly builds:

- Computes version as `{BASE_VERSION}-nightly.{YYYYMMDD}.{SHA}`
- Sets `NIGHTLY_BUILD=true` which disables the GitHub release (artifacts only)
- Deletes and recreates the `nightly` Git tag and GitHub Release (prerelease, not `latest`)
- Same multi-platform and multi-architecture support as release builds

### Docker Build (`docker-build.yml`)

Manually triggered workflow for building Docker images independently of releases.

## Local Development Commands

| Command | Purpose |
|---------|---------|
| `make deps` | Download and verify Go modules |
| `make generate` | Run `go generate ./...` |
| `make build` | Build `picoclaw` for current platform |
| `make build-launcher` | Build Web UI launcher (builds frontend if needed) |
| `make build-launcher-tui` | Build TUI launcher |
| `make build-all` | Cross-compile for all supported platforms |
| `make build-pi-zero` | Build for Raspberry Pi Zero 2 W (32-bit and 64-bit) |
| `make lint` | Run `golangci-lint` |
| `make fmt` | Auto-fix linting issues |
| `make vet` | Static analysis |
| `make test` | Run tests (Go + web) |
| `make check` | Full pre-commit check: `deps + fmt + vet + test` |
| `make docker-build` | Build minimal Alpine-based Docker images |
| `make docker-build-full` | Build full-featured Docker image with Node.js 24 |

## Build Tags

| Tag | Purpose |
|-----|---------|
| `goolm` | Use JSONL (JSON lines) memory store (default, enabled) |
| `stdjson` | Use standard JSON encoding instead of JSONL |
| `whatsapp_native` | Include native WhatsApp (whatsmeow) support; larger binary |

Build command examples:
```bash
go build -tags goolm,stdjson     # Default
go build -tags goolm,whatsapp_native  # With WhatsApp native
```

## Docker

### Images

| Image | Base | Purpose |
|-------|------|---------|
| `docker.io/sipeed/picoclaw:latest` | Alpine 3.23 | Minimal agent/gateway |
| `docker.io/sipeed/picoclaw:launcher` | (multi-stage) | Web UI launcher |
| `docker.io/sipeed/picoclaw:nightly` | Alpine 3.23 | Nightly builds |
| `docker.io/sipeed/picoclaw:nightly-launcher` | (multi-stage) | Nightly launcher |

### Docker Compose Profiles

| Profile | Service | Command |
|---------|---------|---------|
| `agent` | `picoclaw-agent` | One-shot query: `docker compose ... run --rm picoclaw-agent -m "Hello"` |
| `gateway` | `picoclaw-gateway` | Long-running bot: `docker compose ... --profile gateway up` |
| `launcher` | `picoclaw-launcher` | Web UI: `docker compose ... --profile launcher up` |

### Dockerfiles

| File | Purpose |
|------|---------|
| `Dockerfile` | Standard Alpine-based minimal image (2-stage build) |
| `Dockerfile.full` | Full MCP support with Node.js 24 |
| `Dockerfile.goreleaser` | Used by GoReleaser for official releases |
| `Dockerfile.goreleaser.launcher` | Used by GoReleaser for launcher images |
| `Dockerfile.heavy` | Larger image variant |

## Code Quality Gates

All PRs must pass:
1. `golangci-lint` (v2.10.1) with `goolm,stdjson` build tags
2. `govulncheck` security scan
3. All `go test` tests

Merge requirements (per CONTRIBUTING.md):
- CI passes
- At least one maintainer approval
- All review threads resolved
- Complete PR template (including AI disclosure)
- Squash merge to `main`

## Release Process

1. Maintainer triggers `release.yml` workflow with version tag (e.g., `v0.2.3`)
2. `create-tag` job creates and pushes the Git tag
3. `release` job runs GoReleaser:
   - Builds all binaries, archives, packages
   - Pushes Docker images to ghcr.io and docker.io
   - Creates GitHub Release with artifacts
4. `upload-tos` job optionally uploads to Volcengine TOS
5. Release branch (`release/x.y`) cut from `main` for stable releases
