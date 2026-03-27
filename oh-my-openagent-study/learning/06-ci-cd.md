# CI/CD: oh-my-openagent

## Workflow Overview

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `ci.yml` | push (master, dev), PR | Run tests, typecheck, build |
| `publish.yml` | workflow_dispatch | Publish main package to npm |
| `publish-platform.yml` | workflow_call, dispatch | Build and publish 11 platform binaries |
| `lint-workflows.yml` | push, PR | Lint GitHub workflow files |
| `cla.yml` | pull_request | Contributor License Agreement enforcement |
| `sisyphus-agent.yml` | schedule (daily) | Daily automated agent task |
| `refresh-model-capabilities.yml` | schedule (weekly) | Refresh model capabilities |

## CI Pipeline (ci.yml)

### Jobs

```
┌─────────────────────┐
│   block-master-pr    │  Block PRs targeting master branch
└──────────┬──────────┘
           │ (only on PRs)
           ▼
┌─────────────────────┐     ┌──────────────┐
│       test          │────▶│   build      │
└──────────┬──────────┘     └──────┬───────┘
           │                       │
           │                       ▼
           │                ┌──────────────┐
           │                │ draft-release│
           │                │ (dev branch) │
           │                └──────────────┘
┌──────────┴──────────┐
│    typecheck         │
└─────────────────────┘
```

### Test Strategy

Tests are split into two phases due to module cache pollution from `mock.module()`:

**Phase 1: Mock-heavy tests (isolated)**
```bash
bun test src/plugin-handlers
bun test src/hooks/atlas
bun test src/hooks/compaction-context-injector
bun test src/features/tmux-subagent
bun test src/cli/doctor/formatter.test.ts
bun test src/cli/doctor/format-default.test.ts
bun test src/tools/call-omo-agent/sync-executor.test.ts
bun test src/tools/call-omo-agent/session-creator.test.ts
bun test src/tools/session-manager
bun test src/features/opencode-skill-loader/loader.test.ts
bun test src/hooks/anthropic-context-window-limit-recovery/recovery-hook.test.ts
bun test src/hooks/anthropic-context-window-limit-recovery/executor.test.ts
```

**Phase 2: Remaining tests**
```bash
bun test bin script src/config src/mcp src/index.test.ts \
  src/agents src/shared \
  src/cli/run src/cli/config-manager src/cli/mcp-oauth \
  ... (extensive list of directories)
```

### Build Verification

After build, outputs are verified:
- `dist/index.js` must exist
- `dist/index.d.ts` must exist

Auto-commit schema changes on master branch push.

## Publish Pipeline (publish.yml)

### Flow

```
workflow_dispatch (bump: patch|minor|major or version: override)
         │
         ▼
┌─────────────────────┐
│   test + typecheck  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   publish-main       │──────▶ Publish oh-my-opencode to npm
│                      │──────▶ Publish oh-my-openagent (alias) to npm
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  trigger-platform    │  (if skip_platform != true)
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│      release        │──────▶ Create GitHub release
│                      │──────▶ Delete draft release
│                      │──────▶ Merge to master
└─────────────────────┘
```

### Version Management

1. Fetches latest version from npm registry
2. Calculates new version based on bump type (major/minor/patch)
3. Supports version override for pre-release versions
4. Updates version in package.json and all 11 platform packages
5. Publishes with optional dist tag (e.g., beta, next)

### Dual Package Publishing

The same codebase publishes two npm packages:
- `oh-my-opencode` - primary package
- `oh-my-openagent` - alias package (renamed on publish)

Both published with same version, same artifacts.

### npm Provenance

Uses npm provenance for transparency and security:
```bash
npm publish --access public --provenance
```

## Platform Binary Publishing (publish-platform.yml)

### Platform Matrix

11 platforms built in parallel (max-parallel: 11):
- darwin-arm64, darwin-x64, darwin-x64-baseline
- linux-x64, linux-x64-baseline, linux-arm64
- linux-x64-musl, linux-x64-musl-baseline, linux-arm64-musl
- windows-x64, windows-x64-baseline

### Build Strategy

**Windows builds on windows-latest** (avoids bun cross-compile segfaults)

**Baseline builds** pre-download compile target to cache for faster builds.

### Artifact Flow

```
Build binary ──▶ Compress (tar.gz or 7z) ──▶ Upload artifact
                                              │
                                              ▼
                                    Download artifact
                                              │
                                              ▼
                                    Extract & Publish
                                              │
                                              ▼
                                    npm publish (2 packages)
```

### Skip Detection

Before building/publishing, checks if version already exists on npm for both:
- `oh-my-opencode-{platform}`
- `oh-my-openagent-{platform}`

If both exist, skips entirely to avoid redundant work.

## GitHub Actions Configuration

### Permissions

```yaml
permissions:
  contents: write    # For releases, commits
  id-token: write     # For npm provenance
  actions: write      # For triggering other workflows
```

### Concurrency

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

Prevents redundant runs when new commits are pushed.

### OIDC Provenance

All npm publishes use OpenID Connect (OIDC) for provenance:
- Requires `id-token: write` permission
- Uses `NODE_AUTH_TOKEN` secret
- Configured with `NPM_CONFIG_PROVENANCE: true`

## Release Cadence

| Aspect | Value |
|--------|-------|
| Latest Release | v3.13.1 (2026-03-25) |
| Total Releases | 60+ in v3.x line |
| Commits (March 2026) | 907 |
| Commits (today) | 64 |
| Releases per Week | Multiple |

### Release Naming

Releases use semantic versioning with notable codenames:
- v3.5.0 - "Atlas Trusts No One"

## Contributing Workflow

1. Fork and create branch from `dev` (NOT master)
2. CI runs automatically on push/PR
3. All PRs to `dev` branch
4. Releases triggered by maintainers via `workflow_dispatch`
5. Direct `bun publish` prohibited (OIDC provenance issues)

### PR Requirements

- `bun run typecheck` must pass
- `bun run build` must succeed
- Tested locally with OpenCode
- No version changes in `package.json`
