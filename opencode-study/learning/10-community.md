# Community

## Project Overview

**OpenCode** is an open-source AI coding agent maintained by **Anomaly** (https://anoma.ly). It provides a provider-agnostic, locally-running AI coding assistant with a terminal-based interface.

**Repository**: https://github.com/anomalyco/opencode

**License**: MIT

## Governance

### Organization

- **Owner**: Anomaly
- **Default branch**: `dev` (not `main`)
- **Main branch**: `dev` for development

### Maintainer Team

Core maintainers have write access and manage:
- Pull requests
- Issues
- Vouch system
- Releases

### Trust System (Vouch)

OpenCode uses **[vouch](https://github.com/mitchellh/vouch)** to manage contributor trust.

**File**: `.github/VOUCHED.td`

**Mechanics**:
- **Vouched users**: Explicitly trusted contributors
- **Denounced users**: Explicitly blocked (issues/PRs auto-closed)
- **Everyone else**: Normal participation (no vouch required)

**Commands** (for vouched maintainers):
```
vouch           -- Vouch for issue author
vouch @username -- Vouch for specific user
denounce        -- Denounce issue author
denounce @username <reason> -- Denounce with reason
unvouch / unvouch @username -- Remove from list
```

**Denouncement Policy**:
Reserved for:
- Repeated low-quality AI-generated contributions
- Spam
- Bad faith behavior

Not used for disagreements or honest mistakes.

### Issue Requirements

All issues MUST use templates:
- **Bug report** — requires description
- **Feature request** — requires checkbox + description
- **Question** — requires question text

Blank issues auto-close after 2 hours. Issues may be flagged for:
- Not using a template
- Empty/placeholder fields
- AI-generated walls of text
- Missing meaningful content

### PR Requirements

**Issue-First Policy**: All PRs must reference an existing issue (use `Fixes #123` or `Closes #123`)

**PR Title Convention**:
```
<type>(<scope>): <description>

Types: feat, fix, docs, chore, refactor, test
Scopes: app, desktop, opencode, etc.
```

**PR Content**:
- Small, focused changes
- Explain issue and fix
- Screenshots/videos for UI changes
- How to verify for logic changes
- No AI-generated walls of text

**Contribution Areas** (explicitly welcomed):
- Bug fixes
- Additional LSPs/Formatters
- LLM performance improvements
- New provider support
- Environment-specific fixes
- Missing standard behavior
- Documentation improvements

**UI/Core Features**: Require design review with core team before implementation.

## Contributor Health

### Growth Metrics

From STATS.md (as of 2026-01-29):
- **GitHub Downloads**: 7,815,471
- **npm Downloads**: 2,374,982
- **Total**: 10,190,453

Growth pattern shows exponential adoption with occasional viral spikes (e.g., +247K on Jan 6, 2026).

### Vouched Contributors

Current vouch list (`.github/VOUCHED.td`) includes:
- `adamdotdevin`
- `edemaine`
- `fwang`
- `iamdavidhill`
- `jayair`
- `kitlangton`
- `kommander`
- `r44vc0rp`
- `rekram1-node`
- `thdxr`

Notable denouncements:
- `agusbasari29` — AI PR slop
- `atharvau` — AI review spamming
- `danieljoshuanazareth` — (denounced)
- `florianleibert` — (denounced)
- `opencode2026` — (denounced)
- `spider-yamet` — clawdbot/llm psychosis, spam pinging
- `OpenCodeEngineer` — bot that spams issues

## Release Cadence

### Channels

| Branch | Channel | Notes |
|--------|---------|-------|
| `dev` | Development | Bleeding edge |
| `beta` | Beta | Separate `opencode-beta` repo |
| `production` | Stable | Production releases |
| `snapshot-*` | Snapshot | Testing specific commits |

### Versioning

Automated via `./script/version.ts`:
- Semantic versioning (major.minor.patch)
- GitHub releases created automatically
- Tags pushed to trigger release workflows

### Artifacts

- **CLI**: npm, Homebrew, AUR, Scoop, Chocolatey, portable script
- **Desktop**: DMG (macOS Intel/ARM), EXE (Windows), DEB/RPM/AppImage (Linux)
- **GitHub Action**: `anomalyco/opencode/github@latest`

## Community Channels

### Discord

**Invite**: https://discord.gg/opencode

Primary community discussion, support, and announcements.

### X (Twitter)

**Handle**: @opencode

Announcements and updates.

## Policies

### Security

- **No sandbox**: Permission system is UX, not security isolation
- **Server mode**: Opt-in, requires `OPENCODE_SERVER_PASSWORD`
- **Reporting**: GitHub Security Advisories or security@anoma.ly
- **SLA**: 6 business days for acknowledgement

### AI-Generated Content

**Not accepted** for security reports (auto-ban if submitted).

AI-generated PR descriptions and issues without meaningful content may be closed.

### Third-Party Projects

Projects using "opencode" in name must clarify they are not affiliated:

> If you are working on a project that's related to OpenCode and is using "opencode" as part of its name, for example "opencode-dashboard" or "opencode-mobile", please add a note to your README to clarify that it is not built by the OpenCode team and is not affiliated with us in any way.

## Issue Labels

Common labels for contributors:
- `help wanted` — Seeking contributions
- `good first issue` — Beginner-friendly
- `bug` — Bug reports
- `perf` — Performance improvements

## Related Projects

### models.dev

New AI providers should be added to:
https://github.com/anomalyco/models.dev

Most provider support does not require OpenCode code changes.

### GitHub Action

OpenCode can be used as a GitHub Action for PR reviews, issue explanations, and code fixes. Install via:

```bash
opencode github install
```

Or manual setup with workflow file.

### VSCode Extension

Available at `sdks/vscode/` for VSCode integration.
