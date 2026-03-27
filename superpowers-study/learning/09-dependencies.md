# Dependencies

## Dependency Philosophy

**Minimal Dependencies**: The core superpowers skill library has **zero runtime dependencies**. Skills are Markdown documentation files consumed directly by agent runtimes.

This is an intentional design choice:
- Skills must work across all agent platforms (Claude Code, Codex, OpenCode, Gemini)
- No build step required
- No dependency conflicts between platforms
- Skills can be updated via git pull without npm install

## Runtime Dependencies

### Core Package
```json
// package.json (root)
{
  "name": "superpowers",
  "version": "5.0.6",
  "type": "module",
  "main": ".opencode/plugins/superpowers.js"
}
```
**Core dependencies**: None

### Brainstorm Server
```json
// tests/brainstorm-server/package.json
{
  "name": "brainstorm-server-tests",
  "version": "1.0.0",
  "dependencies": {
    "ws": "^8.19.0"
  }
}
```

| Dependency | Version | Purpose | License |
|------------|---------|---------|---------|
| `ws` | ^8.19.0 | WebSocket server for brainstorming | MIT |

**Note**: The brainstorm server is only used for testing. The actual brainstorming functionality runs as shell scripts.

## Development Dependencies

### Test Dependencies
- Node.js (for brainstorm server tests)
- Shell (bash/sh) - for integration tests
- Git (for version control, symlinks)

### Platform-Specific
- **Windows**: Git for Windows (provides bash.exe, cygpath)
- **Unix**: Standard bash/shell

## Package Lock Files

| File | Status |
|------|--------|
| `tests/brainstorm-server/package-lock.json` | Present (v3 lockfile) |

## Dependency Update Policy

**Automatic Updates**:
- Skills update when plugin updates (marketplace)
- Codex/Git: `git pull` fetches latest

**Manual Updates**:
```bash
# Claude Code
/plugin update superpowers

# Gemini CLI
gemini extensions update superpowers

# Codex
cd ~/.codex/superpowers && git pull
```

## Third-Party Services

### Marketplace Integrations
| Platform | Service | Connection |
|----------|---------|------------|
| Claude Code | Official Plugin Marketplace | HTTPS to claude.com |
| Claude Code Alt | Custom marketplace | GitHub releases |
| Cursor | Plugin Marketplace | opencode.ai |
| OpenCode | Bun plugin registry | GitHub git+https |
| Gemini CLI | Extensions registry | GitHub |

### External URLs
| Purpose | URL |
|---------|-----|
| GitHub Repository | https://github.com/obra/superpowers |
| Issue Tracker | https://github.com/obra/superpowers/issues |
| Marketplace Repo | https://github.com/obra/superpowers-marketplace |
| Discord | https://discord.gg/Jd8Vphy9jq |
| Sponsor | https://github.com/sponsors/obra |

## Skill Dependencies

Skills do not depend on each other at runtime. Cross-skill references use explicit markers:
- `**REQUIRED SUB-SKILL:** Use superpowers:test-driven-development`
- `**REQUIRED BACKGROUND:** You MUST understand superpowers:systematic-debugging`

This allows skills to be loaded independently by agent runtimes.

## Agent Platform Compatibility

| Platform | Skill Loading | Dependencies |
|----------|--------------|--------------|
| Claude Code | `Skill` tool | None |
| Codex | Native skill discovery | None |
| OpenCode | Plugin system | Bun |
| Gemini CLI | Extension loading | None |

## Transitive Dependencies

**Minimal attack surface**:
- Core: No npm dependencies
- Brainstorm tests: Only `ws` (MIT licensed, no known vulnerabilities)

## Dependency File Inventory

| Path | Dependencies |
|------|-------------|
| `package.json` | None (empty deps) |
| `tests/brainstorm-server/package.json` | `ws` |
| `tests/brainstorm-server/package-lock.json` | `ws` locked |
| `gemini-extension.json` | None |
| `.claude-plugin/plugin.json` | None |
| `.cursor-plugin/plugin.json` | None |
| `.opencode/INSTALL.md` | None (docs only) |

## Security Considerations

1. **No npm dependencies in core**: Eliminates supply chain attack surface
2. **Git-based distribution**: Changes are reviewable via git history
3. **MIT licensed dependencies**: Permissive licensing
4. **No secret storage**: Skills don't require API keys
5. **Cross-platform hooks**: Use fixed paths for Git Bash on Windows
