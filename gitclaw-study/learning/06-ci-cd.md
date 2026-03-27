# CI/CD

## Overview

GitClaw uses GitHub Actions for automated publishing to npm. The pipeline is triggered on version tags and handles the complete build-test-publish workflow.

## Pipeline Architecture

### Publish Workflow

**File**: `.github/workflows/publish.yml`

```yaml
on:
  push:
    tags:
      - "v*"
```

**Trigger**: Semantic version tags (`v1.0.0`, `v1.1.6`, etc.)

### Pipeline Steps

| Step | Action | Purpose |
|------|--------|---------|
| 1 | `actions/checkout@v4` | Clone repository for build |
| 2 | `actions/setup-node@v4` | Configure Node.js 20 |
| 3 | `npm ci` | Install dependencies |
| 4 | `npm run build` | Compile TypeScript |
| 5 | `npm publish` | Publish to npm registry |

### Environment Configuration

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 20
    registry-url: https://registry.npmjs.org

- run: npm publish --provenance --access public
  env:
    NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

## Security Configuration

### Permissions

```yaml
permissions:
  contents: read
  id-token: write
```

- `contents: read` - Checkout code (minimal permission)
- `id-token: write` - OIDC token for npm provenance attestation

### Secrets Required

| Secret | Purpose | Access |
|--------|---------|--------|
| `NPM_TOKEN` | npm.org authentication | Write to public registry |

## Build Process

### TypeScript Compilation

```bash
npm run build  # tsc && cp src/voice/ui.html dist/voice/
```

1. Run TypeScript compiler (`tsc`)
2. Copy voice UI HTML to dist

### npm Provenance

```bash
npm publish --provenance --access public
```

- `--provenance` - Enable npm provenance (SLSA compliance)
- `--access public` - Publish as public package (free tier compatible)

## Release Flow

```
Developer pushes tag
        |
        v
GitHub Actions detects tag
        |
        v
Checkout + Setup Node 20
        |
        v
npm ci (clean install)
        |
        v
npm run build (compile + copy assets)
        |
        v
npm publish with provenance
        |
        v
Package available on npm
```

## Versioning Strategy

GitClaw uses semantic versioning aligned with git tags:

- Tag format: `v{major}.{minor}.{patch}` (e.g., `v1.1.6`)
- Version in `package.json` must match tag
- Current published version: **1.1.6**

## npm Distribution

| Field | Value |
|-------|-------|
| Registry | https://registry.npmjs.org |
| Access | Public |
| Provenance | Enabled (SLSA) |
| Scope | Unscoped (`gitclaw`) |

## What's Not Automated

Currently, the following are **not** automated:

- Linting/format checks (no CI lint step)
- Test execution in CI (no test step before publish)
- Changelog generation
- GitHub Releases creation
- Pre-release builds (no `next` tag flow)
- Multi-platform testing (Node 20 on ubuntu only)

## Potential Improvements

1. **Add test step** before publish
2. **Add lint step** with ESLint
3. **Add multi-platform matrix** (ubuntu, windows, macos)
4. **Add pre-release workflow** for `next` tags
5. **Auto-generate changelog** via semantic-release
6. **Create GitHub Release** on tag push
