# QMD CI/CD

## CI Pipeline

### Trigger Conditions

- Push to `main` branch
- Pull requests targeting `main` branch

### Test Matrix

| Job | OS | Runtime | Version |
|-----|----|---------|---------|
| `test-node` | Ubuntu, macOS | Node.js | 22, 23 |
| `test-bun` | Ubuntu, macOS | Bun | latest |

### Node.js Tests

```yaml
- name: Tests
  run: npx vitest run --reporter=verbose --testTimeout 60000 test/
  env:
    CI: true
```

Requires: `libsqlite3-dev` (Linux) or `sqlite` via Homebrew (macOS)

### Bun Tests

```yaml
- name: Verify lockfile is up-to-date
  run: bun install --frozen-lockfile

- name: Tests
  run: bun test --timeout 60000 --preload ./src/test-preload.ts test/
  env:
    CI: true
    DYLD_LIBRARY_PATH: /opt/homebrew/opt/sqlite/lib  # macOS
    LD_LIBRARY_PATH: /usr/lib/x86_64-linux-gnu       # Linux
```

Bun tests use `--frozen-lockfile` to prevent stale lockfile issues.

## Publish Pipeline

### Trigger

Runs on version tags: `v*` (e.g., `v2.0.1`)

### Steps

1. Checkout code
2. Setup Bun (latest)
3. Install SQLite (`libsqlite3-dev`)
4. Verify `bun.lock` is up-to-date (`--frozen-lockfile`)
5. Run tests via Bun
6. Setup Node.js 22
7. Build: `npm run build`
8. Publish to npm with provenance: `npm publish --provenance --access public`
9. Extract release notes via `scripts/extract-changelog.sh`
10. Create GitHub release with extracted notes

### Permissions

```yaml
permissions:
  contents: write
  id-token: write  # For npm provenance
```

## Release Process

### Trigger Mechanism

1. Maintainer invokes `/release <version>` skill
2. `scripts/release-context.sh` validates changelog and working directory
3. Pre-push git hook (`scripts/pre-push`) blocks tag pushes unless:
   - `package.json` version matches the tag
   - `CHANGELOG.md` has corresponding entry
   - CI passed on GitHub
4. `scripts/release.sh` renames `[Unreleased]` to `[X.Y.Z] - date`, bumps `package.json`, commits and tags
5. Tag push triggers `publish.yml` workflow

### Changelog Standard

Based on [Keep a Changelog](https://keepachangelog.com/):
- Entries accumulate under `## [Unreleased]`
- Releases formatted as `## [X.Y.Z] - YYYY-MM-DD`
- Sections: `### Changes`, `### Fixes`
- External contributors credited: `#NNN (thanks @username)`
- Numbers required: "2.7x faster", "17x less memory"
- Excludes: internal refactors, dependency bumps, CI changes

### Versioning

Semantic versioning (semver). v2.0.0+ indicates stable SDK API.

## Git Hooks

Installed via `prepare` script:
- `scripts/pre-push`: Validates version/tag alignment before push

## Summary

| Aspect | Implementation |
|--------|----------------|
| CI Platform | GitHub Actions |
| Test Runtimes | Node 22/23 (Ubuntu, macOS), Bun latest (Ubuntu, macOS) |
| Test Framework | Vitest (Node), Bun test (Bun) |
| Lockfile Validation | `--frozen-lockfile` for Bun |
| Publish | npm with provenance, triggered by git tags |
| Release Automation | Tag-based, with changelog validation |
| Versioning | semver, v2.0.0+ stable SDK |
