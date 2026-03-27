# Community Analysis

## Project Overview

**get-shit-done** (GSD) is an open-source project by **TACHES** — a meta-prompting, context engineering, and spec-driven development system for AI coding agents.

## Governance

### Single Maintainer

**TACHES** (@glittercowboy on Discord/GitHub) is the primary maintainer:
- Sole decision-maker for core direction
- Active on Discord and GitHub
- Responds to security issues directly (security@gsd.build)

### No Formal Governance Structure

- No core team beyond maintainer
- No steering committee
- No election process
- No formal membership tiers

## Community Presence

| Platform | Badge/Link | Purpose |
|----------|------------|---------|
| GitHub | [gsd-build/get-shit-done](https://github.com/gsd-build/get-shit-done) | Source, issues, PRs |
| Discord | [discord.gg/gsd](https://discord.gg/gsd) | Discussion, support |
| X/Twitter | [@gsd_foundation](https://x.com/gsd_foundation) | Announcements |
| npm | [get-shit-done-cc](https://www.npmjs.com/package/get-shit-done-cc) | Package distribution |

## Repository Metrics

| Metric | Value |
|--------|-------|
| Current Version | 1.29.0 |
| License | MIT |
| Stars | (visible on GitHub) |
| Forks | (visible on GitHub) |
| Watchers | (visible on GitHub) |
| Issues | Open/closed tracking |
| PRs | Reviewed/merged tracking |

## Funding

From `.github/FUNDING.yml`:
```yaml
github: glittercowboy
```

**GitHub Sponsors only** — no Patreon, Ko-fi, Open Collective, or other platforms.

**$GSD Token:** Dexscreener link in README — indicating potential tokenized governance or economy (Solana).

## Community Ports

GSD has inspired community adaptations for different AI runtimes:

| Project | Platform | Status | Link |
|---------|----------|--------|------|
| gsd-opencode | OpenCode | Active | github.com/rokicool/gsd-opencode |
| gsd-gemini | Gemini CLI | Archived | Original by uberfuzzy |

These were **community-driven ports** before native multi-runtime support was added to core GSD.

## Contributing Patterns

### Contribution Guidelines (CONTRIBUTING.md)

**Core Rules:**
1. **One concern per PR** — Bug fixes, features, refactors separate
2. **No drive-by formatting** — Don't reformat unrelated code
3. **Link issues** — Use `Fixes #123` or `Closes #123`
4. **CI must pass** — All matrix jobs green before merge

### PR Requirements

From pull request template:

```
### Platforms tested
- [ ] macOS
- [ ] Windows (including backslash path handling)
- [ ] Linux

### Runtimes tested
- [ ] Claude Code
- [ ] Gemini CLI
- [ ] OpenCode
- [ ] Codex
- [ ] Copilot
- [ ] N/A (not runtime-specific)
```

### What Gets Accepted

- Bug fixes with tests
- New features with documentation
- Platform-specific patches (Windows path handling)
- Test coverage improvements
- Documentation improvements (translations, corrections)

### What Gets Rejected

- Enterprise patterns (sprints, ceremonies, stakeholder syncs)
- Unnecessary dependencies
- Breaking changes without migration path
- Features violating core philosophy (simplicity over complexity)

## Issue Templates

From `.github/ISSUE_TEMPLATE/`:
- `bug_report.yml` — Bug reports
- `docs_issue.yml` — Documentation issues
- `feature_request.yml` — Feature requests
- `config.yml` — Issue templates configuration

## Security Vulnerability Handling

From `SECURITY.md`:

| Timeline | Response |
|----------|----------|
| Acknowledgment | Within 48 hours |
| Initial assessment | Within 1 week |
| Critical fix | 24-48 hours |
| High fix | 1 week |
| Medium/Low fix | Next release |

**Contact:** security@gsd.build (or DM @glittercowboy on Discord/Twitter)

## Code of Conduct

Not explicitly documented — no `CODE_OF_CONDUCT.md` visible.

## Community Health Indicators

### Activity Patterns

- Regular releases (1.29.0 at time of analysis)
- Active CHANGELOG maintenance
- Responsive issue handling (per Discord/ GitHub presence)
- Multiple languages maintained (5 translations)

### Trust Signals

From README:
> Trusted by engineers at Amazon, Google, Shopify, and Webflow.

(This is a marketing claim — not independently verified.)

### Documentation Quality

- 5 language translations maintained
- Comprehensive command reference
- Architecture documentation for contributors
- Active CHANGELOG

## Multi-Runtime Community

GSD's support for multiple AI coding platforms creates a fragmented community:

| Runtime | Primary Users |
|---------|---------------|
| Claude Code | Primary audience, most active |
| OpenCode | Open-source enthusiasts |
| Gemini CLI | Google ecosystem users |
| Codex | GitHub Copilot users |
| Copilot | GitHub CLI users |
| Cursor | VS Code extenders |
| Windsurf | Codeium users |
| Antigravity | Gemini power users |

Each runtime has slightly different installation and behavior, creating support complexity.

## Community Challenges

### 1. Fragmentation

Supporting 8 runtimes means:
- Testing matrix explosion (each PR tests all 8)
- Documentation must cover runtime-specific quirks
- Different command syntaxes (`/gsd:`, `/gsd-help`, `$gsd-help`)

### 2. Solo Maintenance

Single maintainer creates:
- Bottleneck for reviews/decisions
- Risk if maintainer becomes unavailable
- No community governance for contested decisions

### 3. Tokenized Future?

$GSD token on Solana suggests:
- Potential governance token
- Community-driven development funding
- But also speculation/investor community

## What Works Well

1. **Clear Philosophy** — Anti-enterprise, simplicity-first message resonates
2. **Comprehensive Documentation** — 5 languages, detailed references
3. **Security Focus** — Prompt injection scanning, clear vulnerability reporting
4. **CI/Testing Standards** — Cross-platform matrix, coverage requirements
5. **Community Ports** — Shows demand for multi-runtime support

## Improvement Opportunities

1. **Formal Code of Conduct** — None visible
2. **Governance Documentation** — No public decision-making process
3. **Contributing Recognition** — No hall of fame or contributor spotlight
4. **Sustainability Model** — GitHub Sponsors only, token unclear
5. **Community Governance** — No RFC process for major changes

## Version History Pattern

From CHANGELOG (87KB, ~500+ releases):
- Semantic versioning (1.29.0)
- Detailed per-version changes
- Breaking changes noted
- Contributor acknowledgments

## Assessment Summary

| Dimension | Score | Notes |
|-----------|-------|-------|
| Documentation | 8/10 | Comprehensive, multi-language |
| Community Engagement | 6/10 | Discord active, but fragmented |
| Governance | 3/10 | Single maintainer, no formal structure |
| Security | 8/10 | Clear process, proactive scanning |
| Inclusivity | 5/10 | No code of conduct visible |
| Sustainability | 5/10 | GitHub sponsors only, token unclear |

**Overall:** A young, developer-focused community with strong documentation culture but limited formal governance. The multi-runtime support is both a strength (broader reach) and challenge (fragmentation).
