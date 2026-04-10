# QMD Community

## Repository Overview

| Metric | Value |
|--------|-------|
| Repository | github.com/tobi/qmd |
| Owner | Tobi Lutke (Shopify cofounder) |
| License | MIT |
| Type | Personal/side project |
| Latest release | v2.0.1 (2026-03-10) |

## Community Stats

| Metric | Value |
|--------|-------|
| Total commits | ~267 |
| Merged PRs | ~44 |
| Contributors (by email) | ~40 |
| Owner commits | ~282 |

## Governance Model

### Structure

- **Single maintainer**: Tobi Lutke
- **No formal governance**: No CODEOWNERS, MAINTAINERS, or AUTHORS files
- **Informal operation**: Project operates on contributor goodwill and maintainer discretion

### Decision Making

- Owner makes final decisions on feature acceptance
- Breaking changes unlikely given semver discipline
- SDK API declared stable in v2.0.0

## Contributor Patterns

### PR Workflow

1. All contributions via GitHub PRs
2. Contributors add changelog entries under `[Unreleased]`
3. Format: `#NNN (thanks @username)`
4. CI runs tests on Node 22/23 and Bun across Ubuntu and macOS

### Notable External Contributions

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

### Community Engagement Style

- Owner responds with implementation details
- Community bugs fixed quickly (several fixes in v1.1.2 addressed issues from 10+ prior PRs)
- Feature requests often include working implementations
- Contributors credited in changelog

## Issue Types

- Bug reports (platform-specific: Homebrew sqlite-vec paths, Windows WSL paths, case-sensitive filenames)
- Feature requests (intent parameter, collection ignore patterns, multilingual support)
- Environment issues (GPU initialization, sqlite-vec loading)

## Release Process

1. Maintainer invokes `/release <version>` skill
2. Pre-push hook validates changelog and version alignment
3. Release script renames `[Unreleased]` and creates git tag
4. Tag push triggers GitHub Actions publish workflow
5. GitHub Release created with cumulative notes for minor series

### Changelog Standards

Based on Keep a Changelog:
- Numbers required: "2.7x faster", "17x less memory"
- Credit external contributors
- Group by theme, not file
- Exclude internal refactors and CI changes

## MCP Integration Focus

Project targets AI agent workflows:
- MCP server for Claude Desktop integration
- Claude Code plugin (`claude plugin marketplace add tobi/qmd`)
- JSON/files output formats designed for LLM consumption

## Documentation

| Document | Status |
|----------|--------|
| README.md | Comprehensive user guide |
| docs/SYNTAX.md | Grammar specification |
| CHANGELOG.md | Full history with Keep a Changelog |
| CLAUDE.md | Developer conventions |
| skills/release/SKILL.md | Release workflow |
| CONTRIBUTING.md | Does not exist |
| Issue/PR templates | Not visible |

## Summary

| Aspect | Assessment |
|--------|------------|
| Community size | Small but active (~40 contributors) |
| Governance | Single maintainer, informal |
| Contributor recognition | Good (credited in changelog) |
| Bug fix responsiveness | High (quick turnaround) |
| Feature acceptance | Selective (owner discretion) |
| semver discipline | Strong (v2.0.0 stable SDK) |
| Professional practices | Mature (Shopify background influence) |

QMD is a well-maintained single-maintainer project with active community engagement. The owner has established clear, professional development practices despite being a personal project.
