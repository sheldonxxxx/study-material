# Superpowers Project Overview

## Project Purpose

Superpowers is a complete software development workflow framework for AI coding agents. It provides a library of reusable skills, hooks, and commands that enable AI agents to work more effectively on software development tasks through structured collaboration patterns.

## Project Scope

**Repository:** github.com/obra/superpowers
**Version:** 5.0.6
**Type:** Multi-platform plugin supporting Claude Code, Cursor, Codex, OpenCode, and Gemini CLI
**License:** MIT

## Core Components

### Skills Library (14 distinct skills)

**Testing & Quality:**
- `test-driven-development/` - RED-GREEN-REFACTOR methodology
- `systematic-debugging/` - Root cause analysis techniques
- `verification-before-completion/` - Fix validation

**Collaboration:**
- `brainstorming/` - Socratic design refinement with visual companion
- `writing-plans/` - Implementation planning
- `executing-plans/` - Plan execution with checkpoints
- `dispatching-parallel-agents/` - Concurrent subagent workflows
- `requesting-code-review/` - Pre-review checklists
- `receiving-code-review/` - Feedback response
- `using-git-worktrees/` - Branch management
- `finishing-a-development-branch/` - Merge workflow
- `subagent-driven-development/` - Two-stage review SDD

**Meta:**
- `writing-skills/` - Skill creation methodology
- `using-superpowers/` - System introduction

### Visual Brainstorming Subsystem

A unique component featuring a zero-dependency WebSocket server:
- `skills/brainstorming/scripts/server.cjs` - WebSocket server (RFC 6455 compliant)
- `skills/brainstorming/scripts/helper.js` - Browser client script
- `skills/brainstorming/scripts/frame-template.html` - UI frame

### Platform Integration Layer

| Platform | Entry Point |
|----------|-------------|
| Claude Code | `.claude-plugin/plugin.json` + `hooks/session-start` |
| Cursor | `.cursor-plugin/plugin.json` + `hooks/session-start` |
| OpenCode | `.opencode/plugins/superpowers.js` |
| Codex | Instructions via `.codex/INSTALL.md` |
| Gemini CLI | `gemini-extension.json` |

### Command Entry Points

- `commands/brainstorm.md` - Brainstorming workflow
- `commands/execute-plan.md` - Plan execution
- `commands/write-plan.md` - Plan creation

## Architecture Highlights

### Zero-Dependency Philosophy

The project eliminates external dependencies by implementing core functionality from scratch using only Node.js built-ins (`crypto`, `http`, `fs`, `path`). This approach:
- Eliminates supply chain risk
- Reduces 714 vendored files
- Enables single-file auditing
- Achieves instant startup

### Multi-Platform Hook System

Session hooks (`hooks/session-start`) detect platform via environment variables and inject skill context accordingly:
- `CURSOR_PLUGIN_ROOT` - Cursor detection
- `CLAUDE_PLUGIN_ROOT` - Claude Code detection
- Fallback for other platforms

### Test Suite Organization

```
tests/
├── brainstorm-server/       # WebSocket server tests
├── claude-code/            # Claude Code integration tests
├── explicit-skill-requests/ # Skill triggering tests
├── opencode/               # OpenCode platform tests
├── skill-triggering/        # Skill activation tests
└── subagent-driven-dev/     # SDD integration tests
```

## Key Strengths

1. **Zero-dependency WebSocket implementation** - RFC 6455 compliant
2. **Comprehensive test coverage** for WebSocket protocol (40+ test cases)
3. **Detailed PR template** enforcing evaluation and adversarial testing
4. **Clean multi-platform separation** via plugin architecture
5. **Rich collaboration patterns** for AI agent teamwork

## Key Risks

1. **No type safety** - Pure JavaScript with no type checking
2. **No CI enforcement** - PR template requirements human-enforced only
3. **No linting** - Style inconsistencies accumulate
4. **Plugin code untested** - `superpowers.js` lacks automated verification

## Summary

Superpowers provides a comprehensive skill-based framework for AI coding agents. Its architecture prioritizes simplicity, auditability, and multi-platform support over feature richness, with a strong emphasis on collaboration patterns and systematic debugging methodologies.