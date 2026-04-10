OUTPUT_FILE: /Users/sheldon/Documents/claw/superpowers-study/research/02-tech-stack.md

# Tech Stack Analysis: Superpowers

## Project Overview
**Repository:** github.com/obra/superpowers
**Version:** 5.0.6
**Type:** Multi-platform AI coding agent plugin/framework
**License:** MIT

## Language & Runtime

| Component | Language | Runtime | Module System |
|------------|----------|---------|--------------|
| Main Plugin | JavaScript | Node.js | ES Modules (ESM) |
| Brainstorm Server | JavaScript (CommonJS) | Node.js | CommonJS |
| Skill Files | Markdown | N/A | N/A |
| Hooks | Bash | N/A | N/A |

### Root package.json
```json
{
  "name": "superpowers",
  "version": "5.0.6",
  "type": "module",
  "main": ".opencode/plugins/superpowers.js"
}
```

## Dependencies

### Direct Dependencies (Minimal)

**Root package.json:** No production dependencies (empty `dependencies` object)

**Test Dependencies:**
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

### Zero-Dependency Design Philosophy
The project prides itself on minimal dependencies. The brainstorm server (`server.cjs`) implements WebSocket protocol from scratch (RFC 6455) using only Node.js built-in modules:
- `crypto` - SHA-1 hashing for WebSocket handshake
- `http` - HTTP server
- `fs` - File system operations
- `path` - Path manipulation

## Platform Support

Multi-platform plugin architecture with platform-specific plugins:

| Platform | Plugin Location | Hook System |
|----------|----------------|-------------|
| Claude Code | `.claude-plugin/` | `hooks.json` |
| Cursor | `.cursor-plugin/` | `hooks-cursor.json` |
| Codex | `.codex/` | Manual setup |
| OpenCode | `.opencode/` | Plugin API |
| Gemini CLI | `gemini-extension.json` | Native extension system |

## Build System & Tooling

### No Traditional Build System
- **No webpack, rollup, esbuild, or TypeScript compiler**
- **No monorepo tooling** (Nx, Turborepo, Lerna, etc.)
- **No Docker** configuration
- **No CI/CD workflows** (GitHub Actions not configured)

### Deployment Method
Plugin-based installation via platform marketplaces:
- Claude Code: `/plugin install superpowers@claude-plugins-official`
- Cursor: `/add-plugin superpowers`
- Gemini CLI: `gemini extensions install https://github.com/obra/superpowers`

## Key Files Structure

```
superpowers/
├── package.json              # ES Module project (no deps)
├── .claude-plugin/          # Claude Code plugin
│   └── plugin.json          # Plugin manifest
├── .cursor-plugin/           # Cursor plugin
│   └── plugin.json
├── .opencode/                # OpenCode plugin
│   ├── INSTALL.md
│   └── plugins/superpowers.js  # Main plugin entry
├── .codex/                  # Codex setup
│   └── INSTALL.md
├── skills/                  # Core skills library (16 skills)
│   ├── brainstorming/
│   ├── systematic-debugging/
│   ├── test-driven-development/
│   ├── executing-plans/
│   ├── subagent-driven-development/
│   └── ... (11 more)
├── hooks/                   # Session hooks
│   ├── hooks.json          # Claude Code hooks config
│   ├── hooks-cursor.json   # Cursor hooks config
│   ├── session-start       # Bash hook script
│   └── run-hook.cmd        # Hook runner
├── commands/               # Command definitions
│   ├── brainstorm.md
│   ├── execute-plan.md
│   └── write-plan.md
├── agents/                 # Agent prompt templates
│   └── code-reviewer.md
├── tests/
│   └── brainstorm-server/  # WebSocket server tests
│       ├── package.json
│       ├── package-lock.json
│       └── server.test.js
└── docs/                   # Documentation
```

## Interesting Framework Choices

### 1. Skills System
Skills are markdown files with YAML frontmatter:
```markdown
---
name: skill-name
description: When to use this skill
---

# Skill Content
...
```

Skills are discovered dynamically by the plugin system.

### 2. Zero-Dependency WebSocket Server
The `server.cjs` implements WebSocket protocol manually:
- RFC 6455 handshake with `sec-websocket-key`
- Frame encoding/decoding (masked client frames, unmasked server frames)
- Ping/pong handling
- Auto-reconnect logic in client `helper.js`

### 3. Multi-Platform Hook System
Session hooks trigger on keywords (`startup`, `clear`, `compact`) using bash scripts that output JSON for context injection.

### 4. Visual Companion for Brainstorming
The brainstorming skill uses a WebSocket-based live reload system for visual screen updates during design sessions.

## Version History
- Version 5.0.6 (current)
- Detailed changelog in `RELEASE-NOTES.md` (56KB)

## Summary Table

| Aspect | Choice |
|--------|--------|
| **Language** | JavaScript (ESM + CommonJS) |
| **Runtime** | Node.js |
| **Package Manager** | npm |
| **Build Tool** | None (pure plugin) |
| **TypeScript** | No |
| **Testing Framework** | Node.js native (no framework) |
| **CI/CD** | None |
| **Docker** | None |
| **Monorepo** | No |
| **External Dependencies** | 1 (`ws` - only in tests) |
