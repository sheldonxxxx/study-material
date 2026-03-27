# Tech Stack

## Languages & Core Technology

**Primary Language**: Markdown
- Skills are authored in Markdown with YAML frontmatter
- No compilation or build step required
- SKILL.md files are interpreted directly by agent runtimes

**Supporting Languages**:
- **JavaScript/ESM**: Brainstorm server implementation (`skills/brainstorming/scripts/server.cjs`)
- **Shell (bash/sh)**: Hook scripts for session management (`hooks/*.sh`)
- **CMD/batch**: Windows polyglot wrappers (`hooks/*.cmd`) using `: << 'CMDBLOCK'` technique

**Module System**: ESM (`"type": "module"` in package.json)

## Platform Support

### Multi-Platform Architecture
The project uses polyglot shell scripts to achieve cross-platform compatibility:

| Platform | Entry Point | Shell | Notes |
|----------|-------------|-------|-------|
| Windows | `.cmd` wrapper | CMD.exe + Git Bash | Uses `C:\Program Files\Git\bin\bash.exe` |
| macOS | `.sh` scripts | bash/sh | Standard Unix |
| Linux | `.sh` scripts | bash/sh | Standard Unix |

**Polyglot Pattern**: Scripts use `CMDBLOCK` heredoc to be valid in both CMD and bash simultaneously.

### Agent Platform Integrations

| Platform | Integration Method | Config File |
|----------|-------------------|-------------|
| **Claude Code** | Plugin marketplace + hooks | `.claude-plugin/plugin.json` |
| **Claude Code (Alt)** | Custom marketplace | `.claude-plugin/marketplace.json` |
| **Cursor** | Plugin system | `.cursor-plugin/plugin.json` |
| **Codex** | Symlink to `~/.agents/skills/` | `.codex/INSTALL.md` |
| **OpenCode** | Bun plugin system | `.opencode/INSTALL.md` |
| **Gemini CLI** | Extension format | `gemini-extension.json` |

## File Structure

```
superpowers/
├── skills/              # Core skill definitions (Markdown + YAML frontmatter)
│   ├──SKILL.md files with frontmatter (name, description)
│   └── supporting files (scripts, reference docs)
├── agents/              # Agent configurations
├── commands/            # Command definitions
├── hooks/               # Session hooks (cross-platform)
│   ├── hooks.json       # Claude Code hook config
│   ├── hooks-cursor.json
│   ├── *.sh             # Unix shell scripts
│   └── *.cmd            # Windows polyglot wrappers
├── tests/               # Test suites
│   ├── brainstorm-server/    # Node.js websocket tests
│   ├── claude-code/         # Shell-based integration tests
│   ├── opencode/            # OpenCode plugin tests
│   └── ...
├── docs/                # Documentation
└── .claude-plugin/      # Claude Code plugin manifest
```

## Build System

**Minimal Build**: No traditional build system.

- Core skills are Markdown files interpreted at runtime
- Hook scripts are executed directly
- Plugin manifests (JSON) define integration points

**Version**: 5.0.6 (from package.json)

## Testing Infrastructure

| Test Suite | Technology | Location |
|------------|------------|----------|
| Brainstorm Server | Node.js + `ws` library | `tests/brainstorm-server/` |
| Claude Code Integration | Shell scripts | `tests/claude-code/` |
| OpenCode Plugin | Shell scripts | `tests/opencode/` |
| Skill Triggering | Shell scripts | `tests/skill-triggering/` |
| Subagent Development | Shell scripts | `tests/subagent-driven-dev/` |

## Key Technical Decisions

1. **Flat Skill Namespace**: All skills in single `skills/` directory for discoverability
2. **YAML Frontmatter**: Required `name` and `description` fields per agentskills.io specification
3. **No Build Step**: Skills are documentation, not compiled code
4. **Cross-Platform Hooks**: Polyglot `.cmd`/`.sh` pattern avoids platform-specific issues
5. **ESM Module Type**: Core package.json uses `"type": "module"` for modern JavaScript

## Dependencies

**Runtime Dependencies** (minimal):
- `ws` ^8.19.0 (websocket library for brainstorm server only)

**No runtime dependencies for core skills** - the entire skill library is pure documentation consumed directly by agent runtimes.
