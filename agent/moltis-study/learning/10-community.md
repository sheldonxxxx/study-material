# Community

## Contributor Patterns

### Centralized Ownership

| Rank | Contributor | Commits | Percentage |
|------|-------------|---------|------------|
| 1 | Fabien Penso | 1,987 | 94.2% |
| 2 | Claude (AI agent) | 49 | 2.3% |
| 3 | github-actions[bot] | 31 | 1.5% |
| 4 | vanduc2514 | 7 | 0.3% |
| 5 | Artem | 5 | 0.2% |
| 6 | dependabot[bot] | 4 | 0.2% |
| 7 | codspeed-hq[bot] | 3 | 0.1% |
| 8 | thomas7725353 | 2 | 0.1% |

**Key observation**: The repository is strongly centralized with Fabien Penso (founder/owner) responsible for 94%+ of commits. External contributions are minimal but do occur (PRs #484, #58, #433, #479, #480, #481, #465, #478 merged).

### AI Agent Collaboration

The project explicitly embraces AI agent collaboration:
- 49 commits from "Claude" show successful autonomous AI contribution
- `CLAUDE.md` (405 lines) provides comprehensive engineering guidance for AI agents
- Conventional commits + clear patterns enable AI agents to work effectively

### Issue Templates

Five templates enforce structured contributions:
- `bug_report.yml` — expected/actual behavior, reproduction steps
- `feature_request.yml` — motivation, alternatives considered
- `documentation.yml` — docs improvements
- `model_behavior.yml` — model-specific issues
- `config.yml` — configuration issues

**Blank issues disabled** — contributors must use templates.

## Release Cadence

### Versioning Strategy
- **Format**: Date-based (`YYYYMMDD.NN`)
- **Cargo.toml**: Static `0.1.0` with real version injected via `MOLTIS_VERSION` env var
- **Current**: v0.10.x series (March 2026)

### Recent Activity

| Tag | Date | Notes |
|-----|------|-------|
| v0.10.18 | 2026-03-25 | Latest |
| v0.10.17 | 2026-03-05 | |
| v0.9.0 | 2026-02-17 | |
| v0.8.x series | February 2026 | |

**Cadence**: Extremely active — 11 releases in ~3 weeks, multiple releases per week during development.

## Governance

### Maintainer
- **Fabien Penso**: Founder, primary maintainer, 94%+ of commits

### Project Structure
- **Monorepo**: 52 Rust crates + web UI
- **Single owner**: No formal governance committee
- **No CODEOWNERS file**: Code ownership is implicit

### Decision Making
- Single maintainer model enables fast decisions
- No public ROADMAP.md visible
- External PRs reviewed and merged when appropriate

## Documentation Quality

### CLAUDE.md (Exceptional)
405 lines of detailed engineering guidance including:
- Rust idioms and patterns
- Testing requirements (E2E with Playwright)
- Security practices (SSRF protection, secrets handling)
- Database migration patterns
- Release workflow documentation
- Build and validation commands

### CONTRIBUTING.md (Clear)
- Development setup prerequisites
- Conventional commit requirements
- Validation commands (`just format-check`, `just test`, etc.)
- PR checklist (tests, formatting, session sharing)
- Security issue handling via SECURITY.md

### SECURITY.md
Exists with vulnerability disclosure policy.

## How to Contribute

### Prerequisites
1. Rust (nightly-2025-11-30)
2. Node.js 22
3. Docker (for tool sandboxing)
4. `just` task runner

### Setup
```bash
git clone https://github.com/moltis-org/moltis
cd moltis
just ci  # Run full validation
```

### Before PR
```bash
just format-check
just lint
just test
./scripts/local-validate.sh
```

### Commit Format
```
type(scope): description

feat(channels): add WhatsApp video support
fix(auth): correct passkey assertion handling
docs(readme): update installation instructions
```

Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, etc.

### PR Checklist
- [ ] Tests pass (`just test`)
- [ ] Formatting passes (`just format-check`)
- [ ] Clippy passes (`just lint`)
- [ ] E2E tests pass (`just ui-e2e`)
- [ ] Conventional commit format
- [ ] Descriptive PR description

### Areas Needing Contributors
1. **More channel implementations** — Matrix, Signal, iMessage
2. **More LLM providers** — Claude, Gemini, local models
3. **Plugin ecosystem** — Third-party tool integrations
4. **Documentation** — User guides, API docs
5. **Platform packaging** — Flatpak, Snap improvements

## Community Health

### Strengths
1. **Excellent AI agent documentation** — CLAUDE.md enables autonomous AI collaboration
2. **Comprehensive CI/CD** — 6 workflows covering testing, benchmarking, releases
3. **Template-driven issues** — Reduces low-quality reports
4. **Active maintenance** — Multiple releases per week
5. **Conventional commits** — Clean git history
6. **Security-conscious** — SECURITY.md, SSRF protection, zizmor workflow
7. **Multi-platform** — macOS, Linux, Docker, Homebrew

### Concerns
1. **Centralized model** — 94%+ commits from single maintainer
2. **Limited external engagement** — Few non-bot external contributions
3. **No CODEOWNERS** — Unclear code ownership
4. **No public roadmap** — Strategic direction not documented
5. **Complex issue templates** — May discourage casual contributions

### Opportunities
1. **AI agent pipeline** — 49 commits from AI shows viable collaboration model
2. **Growing release cadence** — Accelerating development
3. **Homebrew integration** — Official package manager support
4. **Discord community** — Direct user engagement channel

## External Integrations

| Service | Integration |
|---------|-------------|
| **Codspeed** | Performance benchmarking |
| **Dependabot** | Automated dependency updates |
| **Homebrew** | Official `moltis` formula |
| **Discord** | Community link in issue templates |
| **GitHub Actions** | CI/CD, releases, docs deploy |

## Communication Channels

- **GitHub Issues**: Bug reports, feature requests
- **GitHub Discussions**: Questions, ideas
- **Discord**: Real-time community chat (link in issue template)
- **GitHub Releases**: Changelog and announcements

## Conclusion

Moltis is a **healthy but nascent open source project** in rapid development. The single-maintainer model is sustainable given comprehensive automation and clear patterns. The project demonstrates successful AI agent collaboration (49 AI commits), which may become a model for future open source development. Long-term growth would benefit from more distributed ownership, but current trajectory shows strong engagement from the maintainer.
