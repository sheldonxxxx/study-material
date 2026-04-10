# CI/CD

## Infrastructure

### Runners

- **Linux**: Blacksmith 4-vCPU Ubuntu 24.04 (`blacksmith-4vcpu-ubuntu-2404`)
- **Windows**: Blacksmith 4-vCPU Windows 2025 (`blacksmith-4vcpu-windows-2025`)
- **macOS**: GitHub-hosted `macos-latest`

### Container Images

Prebuilt images for faster CI startup (`packages/containers`):

| Image | Base | Additions |
|-------|------|-----------|
| `base` | Ubuntu 24.04 | Common build tools |
| `bun-node` | base | Bun + Node.js 24 |
| `rust` | bun-node | Rust stable |
| `tauri-linux` | rust | Tauri Linux dependencies |
| `publish` | bun-node | Docker CLI + AUR tooling |

Multi-arch builds use Buildx with QEMU for amd64 + arm64.

## Workflows

### publish.yml

**Trigger**: Push to `ci`, `dev`, `beta`, `snapshot-*` branches; manual dispatch

**Jobs**:

1. **version** — Determines version, creates GitHub release
2. **build-cli** — Compiles standalone CLI executable
3. **build-tauri** — Cross-platform Tauri desktop builds
4. **build-electron** — Cross-platform Electron desktop builds
5. **publish** — Distributes to npm, AUR, Docker Hub

**Tauri Targets**:
- macOS: x86_64-apple-darwin, aarch64-apple-darwin
- Windows: x86_64-pc-windows-msvc, aarch64-pc-windows-msvc
- Linux: x86_64-unknown-linux-gnu, aarch64-unknown-linux-gnu

**Electron Targets**: Same as Tauri

**Signing**:
- macOS: Developer ID Application (via imported certificates)
- Windows: Tauri code signing
- Linux: No signing

### containers.yml

**Trigger**: Push to `dev` when container files change

Builds and pushes multi-arch container images to GHCR.

### deploy.yml

**Trigger**: Push to `dev` or `production` branches

Uses **SST** (3.18.10) with **Pulumi** for infrastructure deployment to Cloudflare.

**Secrets Required**:
- `CLOUDFLARE_API_TOKEN`
- `PLANETSCALE_SERVICE_TOKEN_NAME`
- `PLANETSCALE_SERVICE_TOKEN`
- `STRIPE_SECRET_KEY` (prod or dev based on branch)

### test.yml

**Trigger**: Push to `dev`, PRs, manual dispatch

**Jobs**:

1. **unit** — `bun turbo test` on Linux and Windows
2. **e2e** — Playwright tests on Linux and Windows

Concurrency: Each `dev` commit gets unique run ID; PRs share group and cancel stale runs.

### typecheck.yml

Runs `bun turbo typecheck` across the monorepo.

### vouch-check-*.yml

Automated vouch/denounce management via GitHub Actions bot (`@vouch`).

### docs-locale-sync.yml

Synchronizes documentation translations.

### Other Workflows

| Workflow | Purpose |
|----------|---------|
| `beta.yml` | Beta release process |
| `close-issues.yml` | Auto-close stale issues |
| `close-stale-prs.yml` | Auto-close stale PRs |
| `compliance-close.yml` | Compliance-based closing |
| `daily-issues-recap.yml` | Daily issue summary |
| `daily-pr-recap.yml` | Daily PR summary |
| `duplicate-issues.yml` | Duplicate issue detection |
| `generate.yml` | Code generation |
| `nix-eval.yml` | Nix evaluation |
| `nix-hashes.yml` | Nix hash updates |
| `notify-discord.yml` | Discord notifications |
| `opencode.yml` | CI self-check |
| `pr-management.yml` | PR management |
| `pr-standards.yml` | PR standards enforcement |
| `publish-github-action.yml` | GitHub Action release |
| `publish-vscode.yml` | VSCode extension release |
| `release-github-action.yml` | GitHub Action versioning |
| `review.yml` | Code review automation |
| `sign-cli.yml` | CLI binary signing |
| `stats.yml` | Usage statistics |
| `storybook.yml` | Storybook deployment |
| `sync-zed-extension.yml` | Zed editor extension sync |
| `triage.yml` | Issue triage |

## Version Management

Version bumping via `./script/version.ts`:

```bash
./script/version.ts  # Outputs: version, release, tag, repo
```

Branches map to release channels:
- `dev` → Development builds
- `beta` → Beta channel (releases to `opencode-beta` repo)
- `production` → Production release
- `snapshot-*` → Snapshot builds

## Distribution Channels

### npm

```bash
npm i -g opencode-ai@latest
```

### Homebrew

```bash
brew install anomalyco/tap/opencode  # Recommended
brew install opencode                  # Official formula (slower updates)
```

### Windows

```bash
scoop install opencode
choco install opencode
```

### Linux

```bash
# Arch
sudo pacman -S opencode              # Stable
paru -S opencode-bin                 # AUR latest

# Nix
nix run nixpkgs#opencode
```

### Portable

```bash
curl -fsSL https://opencode.ai/install | bash
```

### Desktop

DMG, EXE, DEB, RPM, AppImage from GitHub Releases

## Build Artifacts

| Artifact | Location | Purpose |
|----------|----------|---------|
| CLI binaries | `packages/opencode/dist/` | Standalone executables |
| Tauri apps | GitHub Releases | Desktop installers |
| Electron apps | `packages/desktop-electron/dist/` | Electron builds |
| Container images | GHCR (`ghcr.io/anomalyco/*`) | CI caching |
