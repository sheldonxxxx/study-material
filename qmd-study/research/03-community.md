# QMD Community Analysis

## Repository Overview

- **Repository:** github.com/tobi/qmd
- **Owner:** Tobi Lutke (tobi@shopify.com)
- **License:** MIT
- **Type:** Personal/side project with growing community adoption

## Repository Stats

Based on git history analysis (since January 2026):

| Metric | Value |
|--------|-------|
| Total commits | ~267 |
| Merged PRs | ~44 |
| Contributors (by email) | ~40 |
| Latest release | v2.0.1 (2026-03-10) |

## Release Process

QMD uses a formal, tool-assisted release workflow documented in `skills/release/SKILL.md`:

### Release Mechanism

1. **Trigger:** `/release <version>` skill invoked by maintainer
2. **Context gathering:** `scripts/release-context.sh` validates changelog, checks working directory
3. **Validation:** Pre-push git hook (`scripts/pre-push`) blocks tag pushes unless:
   - `package.json` version matches the tag
   - `CHANGELOG.md` has a corresponding entry
   - CI passed on GitHub
4. **Release script:** `scripts/release.sh` renames `[Unreleased]` to `[X.Y.Z] - date`, bumps package.json, commits and tags
5. **GitHub Release:** `scripts/extract-changelog.sh` generates cumulative notes for the full minor series (e.g., 1.0.0 through 1.0.5)
6. **CI/CD:** GitHub Actions workflow (`publish.yml`) runs tests, builds, publishes to npm with provenance

### Versioning

Semantic versioning (semver). The project is now at v2.0.0+ indicating stable SDK API.

### Changelog Standard

Based on [Keep a Changelog](https://keepachangelog.com/):

- **Format:** `## [Unreleased]` accumulates entries; `## [X.Y.Z] - YYYY-MM-DD` for releases
- **Structure:** Optional highlights prose + `### Changes` and `### Fixes` sections
- **Writing guidelines:**
  - Explain the why, not just the what
  - Include numbers ("2.7x faster", "17x less memory")
  - Group by theme, not file
  - Credit external contributors: `#NNN (thanks @username)`
- **Excluded:** Internal refactors, dependency bumps (unless user-facing), CI/tooling changes, test additions

## Contributing Guidelines

### How to Contribute

Based on CLAUDE.md and CHANGELOG.md analysis:

1. **PR-based workflow:** All contributions come through GitHub PRs
2. **Changelog-aware:** Contributors are expected to add entries under `[Unreleased]` with ` #NNN (thanks @username)` format
3. **Tested:** CI runs vitest on Node 22/23 and bun test on Bun across Ubuntu and macOS
4. **Frozen lockfile:** CI verifies `bun.lock` is up-to-date with `--frozen-lockfile`

### CI Pipeline

From `.github/workflows/ci.yml`:

- **Node.js tests:** Ubuntu + macOS, Node versions 22 and 23
- **Bun tests:** Ubuntu + macOS, latest Bun
- **Test command:** `npx vitest run --reporter=verbose --testTimeout 60000 test/` (Node), `bun test --timeout 60000 --preload ./src/test-preload.ts test/` (Bun)
- **SQLite required:** Installed via apt-get (Linux) or Homebrew (macOS)
- **Environment:** `CI: true` set during tests

### Community PRs (External Contributions)

The project actively merges community PRs. Notable external contributions include:

| PR | Description | Contributor |
|----|-------------|-------------|
| #310 | GPU init via node-llama-cpp built-in autoAttempt | @giladgd |
| #180 | Intent parameter for query disambiguation | @vyalamar |
| #304 | Collection ignore patterns | @sebkouba |
| #273 | Multilingual embeddings via QMD_EMBED_MODEL | @daocoding |
| #313 | Configurable expansion context | @0xble |
| #286 | MCP multi-session HTTP transport | @joelev |
| #391 | ONNX conversion script | @shreyaskarnik |
| #396 | Embed batching memory fix | @ProgramCaiCai |

### Top Contributors (by commit count)

1. Tobi Lutke (owner) - ~282 commits
2. ~40 external contributors with 1-10 commits each

## Community Engagement Patterns

### Response Style

- Owner responds to issues and PRs with implementation details
- Community bugs are fixed quickly (several fixes in v1.1.2 addressed issues from 10+ prior PRs)
- Feature requests often come with working implementations

### Issue Types Observed

- Bug reports (platform-specific: Homebrew sqlite-vec paths, Windows WSL paths, case-sensitive filenames)
- Feature requests (intent parameter, collection ignore patterns, multilingual support)
- Environment issues (GPU initialization, sqlite-vec loading)

### MCP Integration

QMD targets AI agent workflows:
- MCP server for Claude Desktop integration
- Claude Code plugin (`claude plugin marketplace add tobi/qmd`)
- JSON/files output formats designed for LLM consumption

## Governance Model

### Project Structure

- **Single maintainer:** Tobi Lutke (Shopify cofounder)
- **Informal governance:** No CODEOWNERS, MAINTAINERS, or AUTHORS files
- **No formal governance docs:** Project operates on contributor goodwill and maintainer discretion

### Decision Making

- Owner makes final decisions on feature acceptance
- Breaking changes unlikely given semver discipline
- SDK API declared stable in v2.0.0

### Documentation

- README.md: User-focused with quick start, architecture diagrams, SDK usage
- CLAUDE.md: Developer conventions (use Bun, don't compile, release process)
- CHANGELOG.md: Full history with Keep a Changelog format
- docs/SYNTAX.md: Query document grammar specification
- skills/release/SKILL.md: Release workflow documentation

## Summary

QMD is a well-maintained single-maintainer project with active community engagement. The owner has established clear, professional development practices:

- **Strengths:**
  - Formal release process with changelog validation
  - Comprehensive CI/CD with multi-platform testing
  - Active community bug fixes and contributions
  - Clear documentation for users and developers
  - semver discipline with stable SDK API

- **Areas for potential improvement:**
  - No formal governance documentation (CODEOWNERS, MAINTAINERS)
  - No issue/PR templates visible in repository
  - No contributing.md file
  - Limited visibility into project roadmap

The project demonstrates mature software engineering practices despite being a personal/side project, likely influenced by the maintainer's background at Shopify.