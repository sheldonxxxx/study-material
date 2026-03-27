# OpenCode Community Health Report

## Repository Overview

- **Repository:** `anomalyco/opencode`
- **Default Branch:** `dev` (not `main`)
- **Remote:** `https://github.com/anomalyco/opencode.git`
- **Stars:** Not available via API (rate limited)
- **Latest Release:** v1.3.2 (2026-03-24)

---

## Governance Structure

### CODEOWNERS

The project uses a CODEOWNERS file to designate ownership for specific packages:

| Package | Owner |
|---------|-------|
| `packages/app/` | @adamdotdevin |
| `packages/tauri/` | @adamdotdevin |
| `packages/desktop/src-tauri/` | @brendonovich |
| `packages/desktop/` | @adamdotdevin |

### Core Team

Listed in `.github/TEAM_MEMBERS`:

- @adamdotdevin
- @Brendonovich
- @fwang
- @Hona
- @iamdavidhill
- @jayair
- @jlongster
- @kitlangton
- @kommander
- @MrMushrooooom
- @nexxeln
- @R44VC0RP
- @rekram1-node
- @RhysSullivan
- @thdxr

### Trust System (Vouch)

The project uses [vouch](https://github.com/mitchellh/vouch) to manage contributor trust, maintained in `.github/VOUCHED.td`.

**Vouched contributors (17):**
adamdotdevin, ariane-emory, edemaine, fwang, iamdavidhill, jayair, kitlangton, kommander, r44vc0rp, rekram1-node, thdxr

**Denounced accounts (6):**
- agusbasari29 (AI PR slop)
- atharvau (AI review spamming)
- danieljoshuanazareth
- florianleibert
- opencode2026
- spider-yamet (clawdbot/llm psychosis, spam pinging)
- OpenCodeEngineer (bot that spams issues)

---

## Contributing Guidelines

### Issue Requirements

**All issues must use templates.** The project enforces strict issue templates:

1. **Bug report** - requires description, optional: plugins, version, OS, terminal
2. **Feature request** - requires checkbox verification that feature hasn't been suggested before, plus description
3. **Question** - requires the question text

Blank issues are auto-closed after 2 hours if they don't meet requirements.

### PR Requirements

- **Issue First Policy:** All PRs must reference an existing issue (using `Fixes #123` or `Closes #123`)
- **Conventional commits:** PR titles follow `type(scope?): message` format
- **No AI-generated walls of text:** Long AI-generated descriptions are rejected
- **UI changes:** Must include screenshots or videos
- **Logic changes:** Must explain how the fix was verified
- **Style preferences:** Single-word naming, no `else`, prefer `.catch()` over `try/catch`, avoid `any`

### Contributing Scope

**Accepted contributions:**
- Bug fixes
- Additional LSPs/Formatters
- LLM performance improvements
- New provider support
- Environment-specific quirk fixes
- Documentation improvements

**Requires design review:**
- UI changes
- Core product features

**Help-wanted labels:**
- `help wanted`
- `good first issue`
- `bug`
- `perf`

---

## Security

### Threat Model

SECURITY.md documents a clear threat model:

- **No sandbox:** OpenCode does not sandbox the agent; permissions are a UX feature, not security
- **Server mode:** Opt-in HTTP Basic Auth via `OPENCODE_SERVER_PASSWORD`
- **Out of scope:** Server access (when opted-in), sandbox escapes, LLM provider data handling, MCP server behavior, malicious config files

### Security Reporting

- Uses GitHub Security Advisories
- Email escalation: `security@anoma.ly`
- Response expected within 6 business days
- Explicitly states AI-generated security reports result in automatic ban

---

## Issue Templates

### Bug Report Template Fields
- Description (required)
- Plugins
- OpenCode version
- Steps to reproduce
- Screenshot/share link
- Operating System
- Terminal

### Feature Request Template Fields
- Checkbox: Verified the feature hasn't been suggested before (required)
- Description of enhancement (required)

### Question Template Fields
- Question text (required)

---

## Repository Activity

### Release Cadence

Very active release schedule. Recent releases:

| Version | Date |
|---------|------|
| v1.3.2 | 2026-03-24 |
| v1.3.1 | 2026-03-24 |
| v1.3.0 | 2026-03-22 |
| v1.2.27 | 2026-03-16 |
| v1.2.26 | 2026-03-13 |
| v1.2.25 | 2026-03-12 |

Releases are approximately every 1-3 days.

### Commit Activity

Recent commit activity (last 30 commits, all from 2026-03-25/26):

- Multiple commits per day
- Mix of `fix:`, `feat:`, `chore:`, `refactor:` prefixes
- PR-based workflow (e.g., `fix(opencode): image paste on Windows Terminal (#17674)`)
- Active development on:
  - Effect framework migration
  - Windows stability improvements
  - MCP (Model Context Protocol) lifecycle
  - Git-backed review modes
  - Syncing feature

### Versioning Tags

Two tagging schemes:
- `github-v*.*.*` style (legacy/internal)
- `v*.*.*` style (current releases, starting github-v1.0.0 through v1.3.2)

---

## Documentation

### README Features
- Internationalized in 20+ languages (extensive localization effort)
- Multiple installation methods: curl script, npm, brew, scoop, choco, pacman, aur, mise, nix
- Desktop app available (DMG, EXE, AppImage)
- Discord community link
- NPM package

### AGENTS.md
- Development guidelines specifically for AI agents
- Enforces single-word naming convention (mandatory rule)
- Testing guidance: avoid mocks, run from package directories
- Type checking: use `bun typecheck` from package dirs

---

## CI/CD

- GitHub Actions workflows present (35 workflow files)
- Publish workflow available (`.github/workflows/publish.yml`)
- Python SDK publishing workflow (`.github/publish-python-sdk.yml`)
- Automated PR/commit checks likely present

---

## Community Health Assessment

| Dimension | Status | Notes |
|-----------|--------|-------|
| Governance | Strong | Clear CODEOWNERS, team members listed, vouch system |
| Contribution friction | Low-medium | Issue templates enforced, but clear guidelines, help-wanted labels |
| Security practices | Mature | Clear threat model, security policy, no AI-generated reports |
| Release cadence | Very active | Multiple releases per week |
| Community engagement | Active | Regular commits, PRs from multiple contributors |
| Documentation | Excellent | Multi-language README, AGENTS.md for AI devs |
| Trust/safety | Strict | Vouch system, denounced spammers, AI-generated content policy |

---

## Key Takeaways

1. **Well-structured governance:** CODEOWNERS for package ownership, team members listed, clear trust system
2. **Active development:** Multiple commits daily, frequent releases
3. **Strict quality controls:** Issue templates required, no AI-generated walls of text, conventional commits
4. **Anti-spam measures:** Vouch/denounce system actively maintained to block AI-generated spam
5. **Developer-friendly:** Clear contributing guidelines, AGENTS.md for AI pair programming, good first issues labeled
6. **International:** README available in 20+ languages
