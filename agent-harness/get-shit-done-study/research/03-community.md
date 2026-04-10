# Community Health Report: get-shit-done

## Repository Stats

| Metric | Value |
|--------|-------|
| Stars | 42,324 |
| Forks | 3,422 |
| Open Issues | 488 |
| Open Pull Requests | 69 |
| Latest Release | v1.29.0 |
| Last Push | 2026-03-25 |

## Repository Topics

- claude-code
- context-engineering
- meta-prompting
- spec-driven-development

## Contributing Guidelines

**Status:** Found and comprehensive

The project has a detailed `CONTRIBUTING.md` at the root level covering:

### Pull Request Guidelines
- One concern per PR (bug fixes, features, refactors separate)
- No drive-by formatting unrelated to changes
- Link issues using `Fixes #123` or `Closes #123`
- CI must pass on all matrix jobs (Ubuntu, macOS, Windows x Node 22, 24)

### Testing Standards
- Uses Node.js built-in test runner (`node:test`) and assertion library (`node:assert`)
- **No external test frameworks** (Jest, Mocha, Chai prohibited)
- Required imports specified: `node:test` and `node:assert/strict`
- **Hooks required**: `beforeEach`/`afterEach` for setup/cleanup (try/finally prohibited)
- Must use centralized test helpers from `tests/helpers.cjs`
- Node version compatibility: Node 22 (LTS) and Node 24 (Current)

### Code Style
- **CommonJS only** (`.cjs` files, `require()` not ESM `import`)
- **No external dependencies in core** (gsd-tools.cjs uses only Node.js built-ins)
- **Conventional commits required**: `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `ci:`

### Security Practices
- Path validation using `validatePath()` from `security.cjs`
- No shell injection: use `execFileSync` (array args) over `execSync`
- No `${{ }}` in GitHub Actions `run:` blocks

### File Structure Documentation
The CONTRIBUTING.md documents the project layout:
```
bin/install.js          — Installer (multi-runtime)
get-shit-done/
  bin/lib/              — Core library modules (.cjs)
  workflows/            — Workflow definitions (.md)
  references/           — Reference documentation (.md)
  templates/            — File templates
agents/                 — Agent definitions (.md)
commands/gsd/           — Slash command definitions (.md)
tests/                  — Test files (.test.cjs)
  helpers.cjs           — Shared test utilities
docs/                   — User-facing documentation
```

## CODEOWNERS

**Status:** Present

```
# All changes require review from project owner
* @glittercowboy
```

Single owner model: all changes require review from `@glittercowboy`.

## Issue and PR Templates

**Status:** Not found

- No `.github/ISSUE_TEMPLATE/` directory
- No `.github/PULL_REQUEST_TEMPLATE/` directory
- Contributors rely on implicit conventions (link issues, conventional commits)

## Release/Versioning Approach

**Status:** GitHub Releases with semantic versioning

- Active release cadence: ~2-3 releases per week
- Recent releases: v1.29.0 (2026-03-25), v1.28.0 (2026-03-22), v1.27.0 (2026-03-20)
- Version range: v1.0.1 through v1.29.0
- One experimental release: v1.10.0-experimental.0
- Tags present in git history

## Community Engagement Patterns

### Issue Activity (Last 30 Days)
- 488 open issues
- Very active issue triage: recent issues from 2026-03-26 include multiple community-reported bugs
- Mix of bug reports, feature requests, and cross-platform compatibility issues
- Labels and milestones used for categorization

### Pull Request Activity (Last 30 Days)
- 69 open PRs
- Active merging: multiple merged PRs daily
- Strong community contribution: external contributors submit docs (Korean, Japanese, Portuguese translations), bug fixes, and features
- Mixed states: OPEN, MERGED, CLOSED (some closed without merge)

### Contributor Diversity
Based on git history, the project has:
- Maintainer: @trek-e (primary contributor, merges most PRs)
- Owner: @glittercowboy (CODEOWNERS)
- Contributors from: GitHub, individual emails
- International community: Korean, Japanese, Portuguese documentation contributions

### Git Activity (Last 30 Days)
- 488 commits from unique contributors
- Multiple commits daily
- Recent focus areas:
  - Cross-platform fixes (Windows, Codex, Windsurf, Gemini CLI)
  - Security hardening (prompt injection scanning, secret scanning)
  - Feature additions (agent skill injection, schema drift detection)

## Observations

### Strengths
1. **Comprehensive CONTRIBUTING.md** - detailed testing standards, code style, and security practices
2. **Active maintenance** - daily commits, frequent releases
3. **Strong community engagement** - bug reports and features from multiple contributors
4. **International reach** - documentation in multiple languages
5. **Security-conscious** - dedicated security scanning, clear security guidelines

### Gaps
1. **No issue/PR templates** - contributors lack structured guidance for filing issues or PRs
2. **Single owner model** - CODEOWNERS gives only @glittercowboy review authority
3. **No MAINTAINERS file** - unclear who else has merge rights or responsibilities
4. **No AUTHORS file** - contributor attributions not formally tracked

### Health Assessment

| Dimension | Status |
|-----------|--------|
| Documentation | Good (CONTRIBUTING.md comprehensive) |
| Issue Management | Active (488 open, active triage) |
| PR Review | Active (daily merges) |
| Release Cadence | Excellent (2-3x weekly) |
| Community Growth | Strong (3.4k forks, international contributions) |
| Governance | Minimal (single owner, no maintainer docs) |
