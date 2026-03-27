# CI/CD Pipeline

## Build & Release Process

### Version Management
- **Current Version**: 5.0.6 (from package.json)
- **Version Tracking**: CHANGELOG.md documents all releases with date and change categories (Fixed, Changed, Added)
- **Release Notes**: RELEASE-NOTES.md provides user-facing release summary

### Release cadence
- Regular releases with explicit versioning
- Changelog shows recent release: v5.0.5 (2026-03-17)
- Changes categorized as: Fixed, Changed, Added

## Distribution Channels

### Claude Code Marketplace
- **Primary Channel**: Official Claude plugin marketplace
- **Installation**: `/plugin install superpowers@claude-plugins-official`
- **Alternative**: Custom marketplace (`obra/superpowers-marketplace`)
  - Register: `/plugin marketplace add obra/superpowers-marketplace`
  - Install: `/plugin install superpowers@superpowers-marketplace`

### Cursor Plugin Marketplace
- **Installation**: `/add-plugin superpowers` or search marketplace
- **Config**: `.cursor-plugin/plugin.json`

### OpenCode (Bun)
- **Installation**: Add to `opencode.json` plugin array
- **Auto-update**: Plugin reinstalls from git on each launch
- **Version pinning**: Use `#v5.0.3` syntax

### Codex
- **Installation**: Git clone + symlink
- **Update**: `git pull` in cloned directory
- **Skills update**: Instant through symlink

### Gemini CLI
- **Installation**: `gemini extensions install https://github.com/obra/superpowers`
- **Update**: `gemini extensions update superpowers`

## Automated Testing

### Test Suites

| Test Suite | Purpose | Technology |
|------------|---------|------------|
| `tests/brainstorm-server/` | WebSocket server tests | Node.js, `ws` library |
| `tests/claude-code/` | Claude Code integration | Shell scripts |
| `tests/opencode/` | OpenCode plugin loading | Shell scripts |
| `tests/skill-triggering/` | Skill activation | Shell scripts |
| `tests/subagent-driven-dev/` | Subagent workflow | Shell scripts |
| `tests/explicit-skill-requests/` | Skill invocation | Shell scripts |

### Running Tests
```bash
# Brainstorm server
cd tests/brainstorm-server && npm test

# OpenCode tests
cd tests/opencode && ./run-tests.sh

# Skill triggering
cd tests/skill-triggering && ./run-all.sh
```

## GitHub Infrastructure

### Issue Templates
- `bug_report.md` - Bug reporting with environment info
- `feature_request.md` - Feature requests
- `platform_support.md` - Platform-specific issues
- `config.yml` - Issue template configuration

### Pull Request Template
Required sections enforce quality standards:
- Problem statement
- Change description
- Core library appropriateness check
- Alternatives considered
- Related PRs check
- Environment tested (harness, version, model)
- Evaluation methodology
- Rigor checklist (adversarial testing, human review)

## CI/CD Gaps

### Not Observed
- **No GitHub Actions workflows** in `.github/workflows/`
- **No automated CI pipeline** for PR validation
- **No automated deployment** to marketplaces
- **No linting/formatting enforcement** for Markdown
- **No automated version bumping**

### Manual Release Process
Based on observed artifacts, releases appear to be:
1. Version bumped in `package.json`
2. Changelog updated with changes
3. GitHub release created manually
4. Marketplaces updated manually or via plugin system hooks

## Hooks System

### Session Start Hook
```json
// hooks.json
{
  "hooks": {
    "SessionStart": [{
      "matcher": "startup|clear|compact",
      "command": "${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd session-start"
    }]
  }
}
```

### Cross-Platform Implementation
- Polyglot `.cmd` wrapper detects platform and delegates to appropriate shell
- Windows: Uses Git Bash at fixed path
- Unix: Direct bash execution

## Update Mechanisms

| Platform | Update Method |
|----------|--------------|
| Claude Code (Marketplace) | `/plugin update superpowers` |
| Claude Code (Custom) | `/plugin update superpowers` |
| Cursor | Restart to auto-update |
| OpenCode | Restart to reinstall from git |
| Codex | `git pull` in clone directory |
| Gemini CLI | `gemini extensions update` |
