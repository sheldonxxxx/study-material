# OpenFang Community

## Repository Overview

| Metric | Value |
|--------|-------|
| Stars | 150 |
| Open Issues | 30 |
| Description | Open-source Agent Operating System |
| Topics | agent-framework, ai-agents, llm, mcp, open-source, openclaw, operating-system, rust |
| Latest Release | v0.5.2 (2026-03-26) |
| License | MIT + Apache 2.0 (dual) |

---

## Release Cadence

**Very active development** with rapid releases:

| Version | Date | Notes |
|---------|------|-------|
| v0.1.0 | 2026-02-24 | Initial release |
| v0.5.0 | Recent | Feature release |
| v0.5.1 | Recent | Patch |
| v0.5.2 | 2026-03-26 | Latest (12 bug fixes) |

Approximately **5 releases in ~1 month**, indicating an active launch phase with frequent iteration.

---

## Git Activity

- **Recent commits (2025-01-01 to present)**: 176 commits
- **Last commit**: 2026-03-26
- **Commit frequency**: Daily or near-daily

The project shows **consistent daily commits** with recent activity.

---

## Contributors

### Total Unique Contributors: 10

| Contributor | Commits | Email | Role |
|-------------|---------|-------|------|
| jaberjaber23 | 132 | jaberib647@gmail.com | Primary maintainer |
| Jaber Jaber | 9 | 103749727+jaberjaber23@users.noreply.github.com | Same person (secondary email) |
| dependabot[bot] | 8 | 49699333+dependabot[bot]@users.noreply.github.com | Automated updates |
| Mark B | 3 | mark@vpost.net | External contributor |
| Liu | 2 | lc-soft@live.cn | External contributor |
| Irwin | 2 | irwingem8@hotmail.com | External contributor |
| Frank | 2 | 97429702+tsubasakong@users.noreply.github.com | External contributor |
| Evan Hu | 2 | suzukaze.haduki@gmail.com | External contributor |
| tuzkier | 1 | GitHub auth | External contributor |
| reaster | 1 | GitHub auth | External contributor |

### Concentration Analysis

**Single dominant contributor**: jaberjaber23 with ~75% of all commits

This is typical for a project in early development but represents a **bus factor risk** -- the project's continuity depends heavily on one person.

---

## Governance

### Present

| File | Status | Details |
|------|--------|---------|
| CONTRIBUTING.md | Yes | 372 lines, comprehensive |
| Issue Templates | Yes | Bug Report, Feature Request |
| PR Template | Yes | Present |
| FUNDING.yml | Yes | Present |
| dependabot.yml | Yes | Active |
| GitHub Workflows | Yes | CI + Release |
| Code of Conduct | Yes | Contributor Covenant v2.1 |

### Missing

| File | Status |
|------|--------|
| CODEOWNERS | No |
| MAINTAINERS | No |
| AUTHORS | No |

---

## Community Health Summary

| Dimension | Assessment |
|-----------|------------|
| Activity Level | Very High (daily commits, rapid releases) |
| Contributor Diversity | Low (single dominant contributor) |
| Documentation | Excellent (comprehensive CONTRIBUTING.md) |
| Templates | Good (issue/PR templates present) |
| Governance | Minimal (no CODEOWNERS/MAINTAINERS/AUTHORS) |
| Open Source Maturity | Early-stage (launch phase, v0.5.x) |
| Security Practices | Good (SECURITY.md, cargo audit, TruffleHog) |
| Dependency Hygiene | Good (dependabot active) |

---

## Key Observations

### Strengths

1. **Rapid development pace** -- Multiple releases per week suggests active development and frequent shipping
2. **Professional tooling** -- dependabot, CI workflows, comprehensive CONTRIBUTING.md indicate production-grade project
3. **Responsive to issues** -- Recent commit "fix: resolve 6 more bugs (#845, #844, #823, #767, #802, #816)" suggests active bug triaging
4. **Good documentation culture** -- Step-by-step guides for adding agents, channels, tools

### Concerns

1. **Single maintainer risk** -- 75% of commits from one contributor creates bus factor concern
2. **Early-stage community** -- Low contributor count but active external contributions
3. **No CODEOWNERS** -- No distributed review responsibility
4. **No MAINTAINERS file** -- No clear secondary maintainers identified

---

## External Integrations

| Integration | Count | Examples |
|------------|-------|----------|
| LLM Providers | 20 | Anthropic, OpenAI, Groq, Gemini, DeepSeek, xAI |
| Messaging Channels | 40 | Discord, Slack, WhatsApp, Telegram, +36 more |
| MCP Templates | 25 | Bundled MCP server templates |
| Skills | 60 | Bundled skill marketplace |

---

## Contributing Guidelines Quality

The CONTRIBUTING.md is **comprehensive and well-structured**:

- Development environment setup (Rust 1.75+, Git, Python)
- Build and testing instructions (1,744+ tests)
- Code style guidelines (rustfmt, clippy, doc comments)
- Architecture overview (14-crate workspace)
- Step-by-step guides for adding:
  - New agent templates
  - New channel adapters
  - New tools
- PR process with review requirements
- Commit message conventions
- Code of Conduct (Contributor Covenant v2.1)

---

## Recommendations for Project Health

1. **Add CODEOWNERS** to distribute review responsibility across crate owners
2. **Consider MAINTAINERS file** to identify secondary maintainers
3. **Build more diverse contributor base** as project matures
4. **Promote external contributors** to reviewers/committers
5. **Create AUTHORS file** to acknowledge all contributors

---

## Support Channels

| Channel | Link |
|---------|------|
| GitHub Issues | https://github.com/RightNow-AI/openfang/issues |
| GitHub Discussions | https://github.com/RightNow-AI/openfang/discussions |
| Security Disclosures | SECURITY.md (private vulnerability reporting) |
| Funding | FUNDING.yml (GitHub Sponsors, etc.) |
