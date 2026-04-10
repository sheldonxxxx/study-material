# Hermes Agent Community Health Assessment

**Repository:** [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
**Assessment Date:** 2026-03-27
**Assessor:** Automated community health assessment

---

## Repository Overview

| Metric | Value |
|--------|-------|
| Description | "The agent that grows with you" |
| Owner | NousResearch |
| License | MIT |
| Default Branch | main |

---

## Repository Statistics

| Metric | Value |
|--------|-------|
| **Recent Release** | v2026.3.23 (2026-03-24) |
| **Release Cadence** | Approximately every 5-7 days |
| **Recent Commits (March 2026)** | 48 commits (partial month) |
| **February 2026 Commits** | 2,156 commits |
| **January 2026 Commits** | 454 commits |
| **Total Releases (observed)** | 3 (v2026.3.12, v2026.3.17, v2026.3.23) |

### Release Tags

| Tag | Release Date | Name |
|-----|--------------|------|
| v2026.3.23 | 2026-03-24 | Hermes Agent v0.4.0 |
| v2026.3.17 | 2026-03-17 | Hermes Agent v0.3.0 |
| v2026.3.12 | 2026-03-12 | Hermes Agent v0.2.0 |

**Versioning Approach:** Date-based versioning (YYYY.M.D) with semantic version bumps (v0.2.0, v0.3.0, v0.4.0). Releases are tied to git tags and follow conventional commit messages.

---

## Contributing Guidelines

### Quality of Contributing Documentation

**CONTRIBUTING.md:** 26,908 bytes — Extremely comprehensive

The contributing guide is notably thorough and covers:

1. **Contribution Priorities** — Explicit hierarchy: Bug fixes > Cross-platform compatibility > Security > Performance > New skills > New tools > Documentation

2. **Skill vs Tool Decision Framework** — Clear guidance distinguishing when to add a skill (preferred) versus a tool, with specific criteria including external dependencies, API key management, and binary data handling

3. **Bundled vs Optional vs Hub Skills** — Three-tier system:
   - `skills/` — Broadly useful, bundled by default
   - `optional-skills/` — Official but not universally needed
   - Skills Hub — Community-contributed specialized skills

4. **Development Setup** — Detailed prerequisites (Python 3.11+, uv, Node.js 18+), installation steps, configuration, and verification commands

5. **Architecture Overview** — Core loop diagram, self-registering tool pattern, session persistence, provider abstraction

6. **Code Style** — PEP 8 with practical exceptions, error handling guidelines, cross-platform requirements

7. **Security Considerations** — Shell injection prevention, sudo password piping, dangerous command detection, cron prompt injection protection, skills guard, code execution sandboxing

8. **PR Process** — Branch naming convention, pre-submission checklist, commit message format (Conventional Commits)

9. **Cross-Platform Compatibility** — Explicit rules for termios/fcntl handling, file encoding, process management, path separators

### PR Template

The PR template (`.github/PULL_REQUEST_TEMPLATE.md`) is highly structured with:

- What/Why description fields
- Related issue linking (Fixes #)
- Type of change checkboxes (bug fix, feature, security, docs, tests, refactor, new skill)
- Specific changes list with file paths
- How to test section with reproduction steps
- Comprehensive checklist:
  - Code: Read contributing guide, commit message format, duplicate check, tests pass, platform testing
  - Documentation: Updated relevant docs
  - Configuration: Updated cli-config.yaml.example
  - Architecture: Updated CONTRIBUTING.md or AGENTS.md
  - Cross-platform: Considered Windows/macOS impact
  - New skills: Broad usefulness, SKILL.md format, no external dependencies, end-to-end testing

### Issue Templates

**Bug Report (`.github/ISSUE_TEMPLATE/bug_report.yaml`):**
- Requires reproduction steps
- Component and platform categorization
- Hermes version, Python version, OS
- Root cause analysis field (optional)
- Proposed fix field (optional)
- PR willingness checkbox

**Feature Request (`.github/ISSUE_TEMPLATE/feature_request.yaml`):**
- Problem/use case description
- Proposed solution
- Alternatives considered
- Feature type dropdown
- Scope estimation (small/medium/large)
- Contribution willingness checkbox

**Setup Help (`.github/ISSUE_TEMPLATE/setup_help.yaml`):**
- Blank issues enabled with contact links

### GitHub Actions Workflows

| Workflow | Purpose |
|----------|---------|
| `tests.yml` | Test suite execution |
| `docs-site-checks.yml` | Documentation validation |
| `deploy-site.yml` | Documentation site deployment |
| `supply-chain-audit.yml` | Security audit |
| `nix.yml` | Nix package manager integration |

---

## Community Structure

### Communication Channels

| Channel | URL |
|---------|-----|
| Discord | discord.gg/NousResearch |
| GitHub Issues | github.com/NousResearch/hermes-agent/issues |
| GitHub Discussions | github.com/NousResearch/hermes-agent/discussions |
| Skills Hub | agentskills.io |

### Maintenance

- **No CODEOWNERS file** observed
- **No MAINTAINERS file** observed
- **No AUTHORS file** observed
- Community appears to be Nous Research-led with strong contributor involvement

---

## Top Contributors

Based on git commit history:

| Rank | Contributor | Approximate Commits |
|------|-------------|---------------------|
| 1 | teknium1 | ~2,027 |
| 2 | 0xbyt4 | ~176 |
| 3 | Test | ~61 |
| 4 | Erosika | ~26 |
| 5 | teyrebaz33 | ~18 |
| 6 | rovle | ~18 |

**Note:** teknium1 appears to be the primary maintainer. The repository shows a clear single-maintainer pattern with occasional contributions from a small secondary contributor pool.

---

## Release Cadence and Versioning

### Versioning Strategy

- **Format:** Date-based (`vYYYY.M.D`) with semantic versions (`v0.4.0`)
- **Examples:** v2026.3.23, v2026.3.17, v2026.3.12
- **Pattern:** Date-based tags for releases, semantic versions for major milestones

### Release Frequency

Based on observed tags:
- **March 2026:** 3 releases in ~17 days (~5-6 day intervals)
- **Overall velocity:** Extremely high (2,156 commits in February 2026 alone)

### Versioning Approach

The project uses a dual versioning system:
1. Semantic version for major features (v0.2.0, v0.3.0, v0.4.0)
2. Date-based tags for continuous delivery (v2026.3.12, v2026.3.17, v2026.3.23)

---

## Community Engagement Patterns

### Strengths

1. **Exceptional Documentation:** 27KB contributing guide with architecture diagrams, security guidelines, and detailed contribution decision frameworks

2. **Structured Contribution Process:** Comprehensive PR template with explicit checklists covering code quality, testing, documentation, and cross-platform compatibility

3. **Multi-Tier Skills Ecosystem:** Clear distinction between bundled, optional, and hub skills enables community contributions while maintaining quality

4. **Security-First Mindset:** Detailed security considerations in CONTRIBUTING.md, supply-chain-audit workflow, explicit security fix type in PR template

5. **Active Development:** Extremely high commit velocity (2,000+ commits/month) indicates very active maintenance

6. **Clear Issue Templates:** Structured bug reports and feature requests with required fields improve issue quality

### Areas of Concern

1. **Contributor Concentration:** Single maintainer (teknium1) with limited secondary contributors suggests potential bus factor risk

2. **No Formal Governance:** No CODEOWNERS, MAINTAINERS, or AUTHORS files to formalize roles and responsibilities

3. **GH Stats Unavailable:** Unable to retrieve current stars, forks, issues, and PR counts via GitHub API (may indicate API limitations)

4. **Recent Project:** Only 3 observed release tags suggests the project is in early/mid stages

---

## Community Health Summary

| Dimension | Assessment |
|-----------|------------|
| **Documentation Quality** | Excellent — Comprehensive contributing guide, architecture docs |
| **Contribution Velocity** | Very High — 2,000+ commits/month |
| **Issue/PR Templates** | Excellent — Structured, detailed, with clear guidance |
| **Release Cadence** | Regular — Every 5-7 days |
| **Versioning** | Clear — Dual semantic + date-based system |
| **Community Channels** | Active — Discord, GitHub Discussions, Skills Hub |
| **Governance** | Minimal — No formal CODEOWNERS/MAINTAINERS |
| **Bus Factor Risk** | Moderate — Heavy reliance on single maintainer |

---

## Recommendations

1. **Add CODEOWNERS file** to formalize code ownership and review requirements
2. **Consider MAINTAINERS file** to identify secondary maintainers
3. **Diversify contributors** through "good first issue" labeling and mentorship programs
4. **Document security release process** for handling sensitive vulnerabilities
