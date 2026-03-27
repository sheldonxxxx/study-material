# Community

## Project

**ZeroClaw** is developed by [zeroclaw-labs](https://github.com/zeroclaw-labs/zeroclaw) and governed by a small team of maintainers.

**Repository:** https://github.com/zeroclaw-labs/zeroclaw

**License:** MIT OR Apache-2.0 (dual)

## Maintainers

Three core maintainers share ownership across all functional areas:

| Maintainer | Role |
|------------|------|
| `@theonlyhennygod` | Primary |
| `@JordanTheJet` | Secondary |
| `@SimianAstronaut7` | Secondary |

### CODEOWNERS Assignment

All project files default to the three-maintainer ownership. Key areas:

```
# Core functional modules
/src/agent/**       @theonlyhennygod @JordanTheJet @SimianAstronaut7
/src/providers/**   @theonlyhennygod @JordanTheJet @SimianAstronaut7
/src/channels/**   @theonlyhennygod @JordanTheJet @SimianAstronaut7
/src/tools/**       @theonlyhennygod @JordanTheJet @SimianAstronaut7
/src/gateway/**    @theonlyhennygod @JordanTheJet @SimianAstronaut7
/src/runtime/**    @theonlyhennygod @JordanTheJet @SimianAstronaut7
/src/memory/**     @theonlyhennygod @JordanTheJet @SimianAstronaut7

# Security, tests, CI/CD
/src/security/**   @theonlyhennygod @JordanTheJet @SimianAstronaut7
/tests/**          @theonlyhennygod @JordanTheJet @SimianAstronaut7
/.github/**        @theonlyhennygod @JordanTheJet @SimianAstronaut7
```

## Contributing

### Branch Model

- **`master`** is the single source-of-truth branch (as of March 2026)
- Contributors fork and create `feat/*` or `fix/*` branches
- All PRs must target `master`
- `main` branch no longer exists (migrated to `master`)

### First-Time Contributors

1. Find issues labeled [`good first issue`](https://github.com/zeroclaw-labs/zeroclaw/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22)
2. Start with **Track A** (docs, tests, chore)
3. Follow fork -> branch -> change -> test -> PR workflow

### Required Checks Before PR

```bash
cargo fmt && cargo clippy && cargo test --locked
```

Or use the CI script:
```bash
./scripts/ci/rust_quality_gate.sh
```

### Pre-push Hook

Enable with:
```bash
git config core.hooksPath .githooks
```

## Collaboration Tracks (Risk-Based)

ZeroClaw uses three tracks based on change risk:

| Track | Scope | Review |
|-------|-------|--------|
| **Track A** | docs/tests/chore, isolated refactors, no security/runtime/CI impact | 1 maintainer + green CI |
| **Track B** | providers/channels/memory/tools behavior changes | 1 subsystem-aware review + validation |
| **Track C** | `src/security/**`, `src/runtime/**`, `src/gateway/**`, `.github/workflows/**`, access-control | 2-pass review + rollback plan |

## Code of Conduct

Adopted from **Contributor Covenant v2.0**.

**Reporting:** violations can be reported to https://x.com/argenistherose

**Enforcement levels:**
1. Correction (unprofessional behavior)
2. Warning (single incident)
3. Temporary Ban (serious violation)
4. Permanent Ban (sustained inappropriate behavior)

## Architecture

ZeroClaw uses a **trait-based pluggable architecture**:

```
src/
├── providers/       # LLM backends     → Provider trait
├── channels/        # Messaging         → Channel trait
├── observability/   # Metrics/logging   → Observer trait
├── runtime/         # Platform adapters → RuntimeAdapter trait
├── tools/           # Agent tools       → Tool trait
├── memory/          # Persistence/brain → Memory trait
└── security/        # Sandboxing        → SecurityPolicy
```

Adding a new integration = implement a trait + register in factory function.

## Documentation System

| Doc | Purpose | Owner |
|-----|---------|-------|
| `docs/README.md` | Canonical docs index | All maintainers |
| `docs/contributing/doc-template.md` | New doc skeleton | All maintainers |
| `CONTRIBUTING.md` | Contributor contract | All maintainers |
| `docs/contributing/pr-workflow.md` | Merge governance | All maintainers |
| `docs/contributing/reviewer-playbook.md` | Review checklist | All maintainers |
| `docs/contributing/ci-map.md` | CI ownership | All maintainers |

## PR Definition of Done

- `CI Required Gate` is green
- Required reviewers approved (CODEOWNERS paths)
- Risk level matches changed paths (`risk: low/medium/high`)
- User-visible behavior, migration, rollback notes complete
- Follow-up TODOs tracked in issues
- `docs/README.md` navigation updated if docs changed

## PR Size Guidelines

Prefer `XS/S/M` sizes. Large work split into stacked PRs.

## High-Volume Collaboration Rules

- One concern per PR (no mixing refactor + feature + infra)
- Small PRs first
- Template mandatory (`.github/pull_request_template.md`)
- Explicit rollback path required
- Security-first review for high-risk paths
- Risk-first triage with labels
- Privacy-first hygiene (redact/anonymize sensitive payloads)

## Agent Collaboration

AI-assisted contributions are welcome and treated as first-class. No requirement to declare AI-vs-human ratio. Contributors remain accountable for understanding the code.

## Security

- **Secret management**: Layered (env vars, `~/.zeroclaw/config.toml`)
- **Encryption**: Secrets encrypted with `chacha20poly1305` AEAD
- **Pre-commit**: `gitleaks` integration (optional)
- **Reporting**: See `SECURITY.md` for vulnerability disclosure

## Public Communications

- **Twitter/X**: Release announcements
- **Discord**: Community discussion and announcements
- **GitHub Discussions**: Technical Q&A

## Release Announcements

Automated via GitHub Actions:
- Twitter (`.github/workflows/tweet-release.yml`)
- Discord (`.github/workflows/discord-release.yml`)

## Community Metrics

The project maintains 40+ translated README files covering major world languages, indicating international adoption.
