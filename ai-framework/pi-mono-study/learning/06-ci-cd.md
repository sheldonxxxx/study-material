# CI/CD

## Workflows

### ci.yml

**Trigger:** Push to main, PRs to main

**Purpose:** Build, check, and test on every change

```yaml
- Checkout
- Setup Node.js 22 with npm cache
- Install system dependencies (libcairo2-dev, libpango1.0-dev, libjpeg-dev, libgif-dev, librsvg2-dev, fd-find, ripgrep)
- npm ci
- npm run build
- npm run check
- npm test
```

**Concurrency:** Cancels in-progress runs for the same ref

### build-binaries.yml

**Trigger:** Git tags (`v*`), manual dispatch

**Purpose:** Cross-platform binary compilation using Bun

```yaml
- Checkout at tag ref
- Setup Bun 1.2.20
- Setup Node.js 22
- Build binaries via ./scripts/build-binaries.sh
- Extract changelog for version
- Create GitHub Release with binaries
```

**Outputs:** `pi-darwin-arm64.tar.gz`, `pi-darwin-x64.tar.gz`, `pi-linux-x64.tar.gz`, `pi-linux-arm64.tar.gz`, `pi-windows-x64.zip`

### pr-gate.yml

**Trigger:** PR opened

**Purpose:** Contributor approval gate -- auto-closes PRs from non-approved contributors

**Logic:**
1. Skip bots (dependabot, etc.)
2. Check if author has admin/maintain/write permission -- allow if yes
3. Check if author is in `.github/APPROVED_CONTRIBUTORS`
4. If OSS weekend active, close with pause message
5. If not approved, close with instructions to open issue first

### approve-contributor.yml

**Trigger:** Manual

**Purpose:** Add approved contributors

### openclaw-gate.yml

**Trigger:** Manual

**Purpose:** OpenClaw integration

### oss-weekend-issues.yml

**Trigger:** Scheduled

**Purpose:** Auto-close issues during OSS weekend

## CI Environment

| Aspect | Value |
|--------|-------|
| Runner | ubuntu-latest |
| Node.js | 22 |
| Package manager | npm |

## Release Process

### Version Management

**Lockstep versioning:** All packages share the same version number.

### Release Commands

```bash
npm run release:patch   # Bug fixes and new features
npm run release:minor   # API breaking changes
```

The release script handles:
1. Version bump across all packages
2. CHANGELOG finalization (convert Unreleased to version)
3. Git commit and tag
4. npm publish to registry
5. Create new Unreleased sections

### Release Cadence

- Every 2-4 weeks
- Recent: v0.62.0 (2026-03-23), v0.61.1, v0.61.0, v0.60.0, v0.59.0

## Testing

| Package | Test Runner | Notes |
|---------|-------------|-------|
| tui | `node --test` | Built-in Node test runner |
| ai | vitest | Unit and integration tests |
| agent | vitest | Unit and integration tests |
| coding-agent | vitest | Unit and integration tests |

**CI test command:** `npm test`

**Skip LLM-dependent tests:** `./test.sh` skips tests requiring API keys

## Binary Build Process

1. Build via Bun: `bun build --compile ./dist/bun/cli.js --outfile dist/pi`
2. Package for each platform using `scripts/build-binaries.sh`
3. Attach to GitHub Release on tag push
