# ZeroClaw Community Health and Metadata

## Repository Identity

| Field | Value |
|-------|-------|
| **Official Repository** | https://github.com/zeroclaw-labs/zeroclaw |
| **Default Branch** | `master` (as of March 2026, `main` was deleted to resolve CI confusion) |
| **License** | Dual: MIT + Apache 2.0 (contributors grant rights under both) |
| **Copyright** | ZeroClaw Labs (2025) |

## Governance Structure

### Code Owners

Three core maintainers share ownership across all functional areas:

| Maintainer | Primary Areas |
|-------------|----------------|
| @theonlyhennygod | All modules, CI/CD, security |
| @JordanTheJet | All modules, CI/CD, security |
| @SimianAstronaut7 | All modules, CI/CD, security |

CODEOWNERS covers:
- All source modules (`/src/**`)
- Security, tests, CI/CD (`.github/**`, `/src/security/**`)
- Documentation and governance (`/docs/**`, `AGENTS.md`, `CONTRIBUTING.md`)
- Configuration files (`Cargo.toml`, `Cargo.lock`)

### Branching Model

```
master (single source of truth)
  └── contributors fork and create feat/* or fix/* branches
       └── PRs target master
```

Recent history shows branch cleanup efforts (deleted stale `main` branch in March 2026 after it caused confusion with 404 errors and broken CI refs).

## Community Guidelines

### Code of Conduct

- **Standard**: Contributor Covenant v2.0
- **Enforcement Contact**: https://x.com/argenistherose (Twitter/X)
- **Scope**: All community spaces, public representation
- **Enforcement Tiers**: Correction (1) -> Warning (2) -> Temporary Ban (3) -> Permanent Ban (4)

### Contributing Guide

Comprehensive 583-line guide (`CONTRIBUTING.md`) covering:

| Section | Key Points |
|---------|------------|
| **Branch Model** | Fork -> `feat/*` or `fix/*` -> PR to `master` |
| **First-Time Contributors** | "good first issue" label, Track A (docs/tests/chore) |
| **Development Setup** | `cargo build`, `cargo test --locked`, pre-push hooks |
| **Collaboration Tracks** | Risk-based: Track A (low), Track B (medium), Track C (high) |
| **PR Definition of Ready** | Template completed, validation run, security impact described |
| **PR Definition of Done** | Green CI, required reviewers, risk label, rollback plan |
| **Commit Convention** | Conventional Commits (`feat:`, `fix:`, `docs:`, etc.) |

### Agent Collaboration

`AGENTS.md` provides first-class support for AI-assisted contributions:
- Risk tiers (Low/Medium/High) classify changes
- Pre-PR validation commands documented
- Anti-patterns explicitly listed (no heavy dependencies, no speculative config flags)
- Agent-assisted PRs treated as first-class, no AI-vs-human ratio required

## Issue Templates

### Bug Report
- Component dropdown (runtime, provider, channel, memory, security, tooling, docs, unknown)
- Severity scale (S0 data loss/security to S3 minor)
- Reproduction steps with bash rendering
- Regression status
- Pre-flight checks (latest branch confirmation, data redaction)

### Feature Request
- Problem statement (required)
- Proposed solution (required)
- Non-goals, alternatives, acceptance criteria
- Architecture impact assessment
- Risk and rollback considerations
- Breaking change dropdown
- Data hygiene checks

## Pull Request Template

Extremely detailed template with 100+ line items:

| Section | Fields |
|---------|--------|
| **Summary** | Problem, why it matters, what changed, scope boundary |
| **Labels** | Risk, size, scope, module, contributor tier |
| **Validation** | `cargo fmt`, `cargo clippy`, `cargo test` evidence |
| **Security** | Permissions, network calls, secrets, filesystem scope |
| **Privacy** | Redaction/anonymization confirmation |
| **Compatibility** | Backward compatibility, migration steps |
| **i18n** | Locale parity for 31 README languages |
| **Rollback** | Fast rollback command, feature flags, failure symptoms |
| **Agent Collaboration** | Tools used, workflow summary, architecture compliance |

## Security

### Security Policy (`SECURITY.md`)

- **Supported Versions**: Only 0.1.x currently marked as supported
- **Reporting**: GitHub Security Advisories (preferred) or private vulnerability reporting
- **Response Timeline**: 48h acknowledgment, 1 week assessment, 2 weeks for critical fixes

### Security Architecture

Defense-in-depth with autonomy levels:
- **ReadOnly**: Read-only, no shell or write access
- **Supervised**: Acts within allowlists (default)
- **Full**: Full access within workspace sandbox

Five sandboxing layers:
1. Workspace isolation
2. Path traversal blocking
3. Command allowlisting
4. Forbidden path list (`/etc`, `/root`, `~/.ssh`)
5. Rate limiting (max actions/hour, cost/day caps)

### Container Security

CIS Docker Benchmark alignment:
- Non-root user (UID 65534 distroless nonroot)
- Minimal base image (`gcr.io/distroless/cc-debian12:nonroot`)
- Read-only filesystem support
- CI enforcement of non-root user verification

## Release Cadence

### Versioning

Semantic versioning with beta pre-release tags:
- **Stable**: `v0.6.2`, `v0.6.1`, `v0.6.0`
- **Beta**: `v0.6.1-beta.637`, `v0.5.9-beta.579`
- **Pattern**: `MAJOR.MINOR.PATCH[-beta.BUILD]`

### Recent Release Activity

| Version | Date | Notable Activity |
|---------|------|-------------------|
| v0.6.3 | 2026-03-26 | Version bump |
| v0.6.2 | 2026-03-24 | Event-triggered automation, routine engine |
| v0.6.1 | 2026-03-24 | |
| v0.6.0 | 2026-03-23 | Major release |
| v0.5.9 | 2026-03-22 | |
| v0.5.6-v0.5.8 | 2026-03-21 | Multiple beta iterations |

Release cadence shows rapid iteration with frequent beta pre-releases (59 total releases in 0.5.x line).

### Release Automation

Workflows for multi-platform publishing:
- `release-stable-manual.yml`: Manual stable releases
- `release-beta-on-push.yml`: Automatic beta on version bump
- `publish-crates.yml`: Cargo crate publication
- `publish-crates-auto.yml`: Crate auto-publish
- `pub-homebrew-core.yml`, `pub-scoop.yml`, `pub-aur.yml`: Package managers
- `discord-release.yml`, `tweet-release.yml`: Community notifications

## Dependency Management

### Dependabot Configuration

Daily updates with PR limits:

| Ecosystem | Schedule | Open PR Limit | Labels | Grouping |
|-----------|----------|---------------|--------|----------|
| Cargo | Daily | 3 | dependencies | All rust (minor+patch) |
| GitHub Actions | Daily | 1 | ci, dependencies | All actions (minor+patch) |
| Docker | Daily | 1 | ci, dependencies | All docker (minor+patch) |

## CI/CD Infrastructure

### Quality Gate Workflow

Runs on every PR to `master`:
- **Lint** (10 min timeout): Formatting check, clippy
- **Test** (30 min timeout): `cargo nextest run --locked`, mold linker
- **Build** (40 min timeout): Multi-platform (Linux x86_64, macOS ARM64, Windows x86_64)

### Additional Workflows

- `ci-run.yml`: General CI orchestration
- `cross-platform-build-manual.yml`: Manual multi-platform builds
- `pr-path-labeler.yml`: Automatic PR labeling by path
- `master-branch-flow.md`: Branch protection and flow documentation

## Contributor Base

### Top Contributors

| Rank | Author | Commits |
|------|--------|---------|
| 1 | Chummy | 832 |
| 2 | argenis de la rosa | 750 |
| 3 | Argenis | 696 |
| 4 | xj | 138 |
| 5 | Will Sarg | 127 |
| 6 | Alex Gorevski | 107 |
| 7 | fettpl | 76 |
| 8 | SimianAstronaut7 | 64 |

**Total Unique Contributors**: 275

Note: Top contributors show heavy concentration (3 accounts with 2200+ combined commits), indicating a core team rather than distributed community. The name variations (Chummy/Chum Yin, argenis/Argenis) suggest multiple accounts or aliases.

## Internationalization

### README Translations

31 language variants available:
- Arabic, Bengali, Chinese (Simplified + Traditional), Czech, Danish, Dutch, English, Finnish, French, German, Greek, Hebrew, Hindi, Hungarian, Indonesian, Italian, Japanese, Korean, Norwegian, Polish, Portuguese, Romanian, Russian, Spanish, Swedish, Thai, Tagalog, Turkish, Ukrainian, Urdu, Vietnamese

### Localization Infrastructure

- `docs/i18n/vi/**`: Vietnamese canonical docs
- `docs/*.vi.md`: Vietnamese compatibility shims
- i18n follow-through tracked in PR template
- Tool descriptions localized across all 31 languages (recently added)

## Observability

### CHANGELOG Status

**Empty** (`CHANGELOG.md` contains only "# Changelog\n")

This is a gap - automated release notes or changelog synchronization may be needed.

### NOTICES File

Complete with:
- Copyright notice
- Official repository statement
- Dual license explanation
- Contributor attribution policy
- Third-party dependency notice
- Verifiable Intent module attribution

## Community Health Assessment

### Strengths

1. **Comprehensive contribution infrastructure**: Detailed guides, templates, CI/CD
2. **Risk-based collaboration**: Three tracks (A/B/C) proportional to change impact
3. **Strong security posture**: Defense-in-depth, security policy, container hardening
4. **Multi-platform support**: Linux, macOS, Windows + multiple package managers
5. **Internationalized**: 31 README languages, localization infrastructure
6. **Active development**: Frequent releases (0.5.x had 59 releases, 0.6.x actively developing)
7. **Automated dependency management**: Dependabot for cargo, actions, docker
8. **Agent-friendly**: First-class AI collaboration support documented

### Weaknesses / Gaps

1. **CHANGELOG.md is empty** - no automated or manual changelog
2. **Concentrated contributor base** - top 3 accounts = majority of commits
3. **Only 0.1.x marked as supported** in security policy (outdated)
4. **No public roadmap** visible in repository
5. **Limited external community engagement** indicators

### Metadata Completeness

| Document | Present | Quality |
|----------|---------|---------|
| CODE_OF_CONDUCT.md | Yes | Good (v2.0, enforcement contact) |
| CONTRIBUTING.md | Yes | Excellent (583 lines, comprehensive) |
| SECURITY.md | Yes | Excellent (detailed architecture) |
| CODEOWNERS | Yes | Good (3 maintainers, full coverage) |
| AGENTS.md | Yes | Excellent (agent collaboration guide) |
| LICENSE (dual) | Yes | Good (MIT + Apache 2.0) |
| NOTICE | Yes | Good (complete attribution) |
| CHANGELOG.md | Yes | **Poor (empty)** |
| Issue Templates | Yes | Excellent (structured, comprehensive) |
| PR Template | Yes | Excellent (100+ line items) |
| Dependabot | Yes | Good (daily, grouped) |
| CI/CD | Yes | Excellent (multi-platform, quality gate) |
