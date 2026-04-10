# OpenClaw Community Health

## Project Overview

- **Repository**: https://github.com/openclaw/openclaw
- **License**: MIT (Copyright 2025 Peter Steinberger)
- **Language**: TypeScript (Node.js 22+, ESM)
- **Current Version**: 2026.3.24

OpenClaw is a multi-channel AI gateway with extensible messaging integrations - "the AI that actually does things." It runs on user devices, in their channels, with their rules.

## Governance Model

**Project Lead**: Peter Steinberger (Benevolent Dictator)

**Maintainer Team** (22 members):
- Shadow (Discord subsystem, community moderation)
- Vignesh (Memory, TUI, IRC, Lobster)
- Jos (Telegram, API, Nix mode)
- Ayaan Zaidi (Telegram, Android app)
- Tyler Yust (Agents, macOS app)
- Mariano Belinky (iOS app, Security)
- Nimrod Gutman (iOS/macOS apps)
- Vincent Koc (Agents, Telemetry, Hooks, Security)
- Val Alexander (UI/UX, Docs, Agent DevX)
- Seb Slight (Docs, Agent Reliability)
- Christoph Nakazawa (JS Infra)
- Gustavo Madeira Santana (Multi-agents, CLI, Performance)
- Onur Solmaz (Agents, ACP integrations)
- Josh Avant (Core, CLI, Gateway)
- Jonathan Taylor (ACP subsystem, CLI tools)
- Josh Lehman (Compaction, Urbit)
- Radek Sienkiewicz (Docs, Control UI)
- Muhammed Mukhthar (Mattermost, CLI)
- Altay (Agents, CLI)
- Robin Waslander (Security, PR triage)
- Tengji Zhang (Chinese model APIs)

## Contributor Landscape

**Top Contributors by Commit Count** (since 2025-01-01):

| Commits | Email | Focus Areas |
|---------|-------|-------------|
| 13,796 | steipete@gmail.com | Core, all areas |
| 1,396 | vincentkoc@ieee.org | Agents, Telemetry, Hooks, Security |
| 409 | vigneshnatarajan92@gmail.com | Memory, TUI, IRC, Lobster |
| 394 | gumadeiras@gmail.com | Multi-agents, CLI, Performance |
| 334 | zaidi@uplause.io | Telegram, Android |
| 292 | Takhoffman | Various |
| 201 | hi@shadowing.dev | Discord, community moderation |
| 156 | hi@obviy.us | Telegram, Android |
| 156 | christoph.pojer@gmail.com | JS Infra |
| 151 | sebslight | Docs, Agent Reliability |

**Community engagement patterns**:
- Regular external contributors submitting bug fixes
- Contributors from diverse backgrounds
- Community members fixing issues across multiple subsystems
- Good representation from various specialties (mobile, backend, security, docs)

## Release Cadence

**Versioning Scheme**: `YYYY.M.D` format (e.g., 2026.3.24)

**Recent releases**:
- **2026.3.24** (latest stable) - March 24, 2026
- **2026.3.24-beta.2** - Beta release
- **2026.3.24-beta.1** - Beta release
- **2026.3.23** - Previous stable

**Cadence characteristics**:
- Multiple beta releases before stable
- Near-weekly stable releases
- Each release contains 10-40+ changes including features, fixes, and breaking changes
- Rapid iteration with stable releases

**Changelog style**:
- Grouped by: Breaking, Changes, Fixes
- Each entry includes PR number and contributor attribution
- Community contributions clearly marked with "Thanks @username"

## Documentation Quality

**Contributing Guidelines** (CONTRIBUTING.md):
- Comprehensive contributing documentation with welcoming tone ("Welcome to the lobster tank!")
- Clear contribution paths:
  - Bugs and small fixes: Open a PR directly
  - New features/architecture: Start a GitHub Discussion or ask on Discord first
  - Refactor-only PRs: Not accepted unless explicitly requested
  - Test/CI-only PRs for known `main` failures: Not accepted

**Development Requirements**:
- Run `pnpm build && pnpm check && pnpm test` before submitting
- Use Codex review tool if available
- Keep PRs focused (one thing per PR)
- **AI/Vibe-coded PRs are explicitly welcomed** and treated as first-class citizens
- Maintainers use a structured PR template

**Issue and PR Templates**:
- Structured bug reports with environment details, steps to reproduce
- Feature requests with use case description
- Comprehensive PR template requiring:
  - 2-5 bullet summary (problem, why it matters, what changed, scope boundary)
  - Root cause analysis for bug fixes
  - Regression test plan
  - User-visible behavior changes
  - Security impact assessment
  - Repro + verification steps

**Security Policy** (SECURITY.md):
- Private vulnerability reporting process
- Required report format (title, severity, impact, affected component, reproduction, demonstrated impact, environment, remediation)
- Extensive list of common false-positive patterns
- Detailed trust model documentation
- Runtime requirements (Node.js 22.12.0+ for security patches)
- Docker security guidance

**CODEOWNERS**: Security-sensitive code review requirements:
- `/SECURITY.md`, `/.github/dependabot.yml`, `/.github/codeql/` require `@openclaw/secops`
- Extensive security-sensitive paths: `src/security/`, `src/secrets/`, `src/config/*secret*.ts`, `src/gateway/*auth*.ts`, `src/agents/*auth*.ts`
- Release workflow requires `@openclaw/openclaw-release-managers`

## Security Processes

**Security features**:
- CODEOWNERS for security-sensitive paths
- Detailed security policy with false-positive guidance
- Clear trust model documentation
- Security-focused review requirements
- `detect-secrets` for secret detection in CI/CD

**Note**: No formal bug bounty program (security policy notes "no budget for paid reports"). Vulnerability disclosure relies on community goodwill.

## Communication Channels

- **Discord**: https://discord.gg/qkhbAGHRBT (active community, `#help` and `#users-helping-users` channels)
- **X/Twitter**: @steipete (Peter Steinberger), @openclaw (project account)
- **Email**: security@openclaw.ai (security reports), contributing@openclaw.ai (maintainer applications)

## Dependency Management

**Dependabot Configuration**:
- npm (root) - Daily, 10 PR limit, grouped production/development
- GitHub Actions - Daily, 5 PR limit
- Swift (macOS app) - Daily, 5 PR limit
- Gradle (Android) - Daily, 5 PR limit
- Docker - Weekly, 5 PR limit
- 2-day cooldown between PR merges

## Project Health Assessment

**Strengths**:
1. Comprehensive documentation with clear expectations
2. Active maintenance with multiple daily commits
3. Rapid release cadence (multiple releases per week)
4. Strong security culture with CODEOWNERS and detailed security policy
5. Community inclusive: AI-coded PRs welcomed
6. Large diverse maintainer team (22+ members)
7. Professional infrastructure: vitest, TypeScript strict, multiple CI/CD pipelines

**Areas for potential improvement**:
1. No formal bug bounty program
2. Vulnerability disclosure relies on community goodwill
3. SECURITY.md is embedded in CONTRIBUTING.md rather than standalone
4. Chinese translations are auto-generated; community translations not accepted

## Community Health Score: 8.5/10

The project demonstrates healthy community dynamics with professional infrastructure, clear governance, active maintenance, and inclusive policies. Minor areas for improvement include formalizing the security reporting infrastructure and potentially expanding the maintainer team for specific areas.
