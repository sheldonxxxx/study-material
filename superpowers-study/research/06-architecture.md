OUTPUT_FILE: /Users/sheldon/Documents/claw/superpowers-study/research/06-architecture.md

# Superpowers Architecture & Design Patterns

## Overview

**Project:** Superpowers - A complete software development workflow for AI coding agents
**Repository:** /Users/sheldon/Documents/claw/reference/superpowers
**Version:** 5.0.6
**Architecture Pattern:** Skills-based plugin system with layered platform abstraction

---

## Architectural Pattern: Skills-Based Plugin System

The fundamental architecture is a **skills-based plugin system** where AI agents invoke discrete, proven workflow templates ("skills") to guide their behavior. Skills are markdown documents with YAML frontmatter that define:

1. **When to invoke** (description field as triggering condition)
2. **How to execute** (step-by-step process with flowcharts)
3. **What to avoid** (rationalization tables, red flags)

This is not a traditional layered architecture. Instead, it follows a **hub-and-spoke pattern** where the skills directory is the central knowledge hub, and platform plugins are adapters that inject skill context into different AI coding environments.

---

## Module Map & Responsibilities

### 1. Platform Integration Layer

**Directory:** `.claude-plugin/`, `.cursor-plugin/`, `.opencode/plugins/`, `.codex/`, `gemini-extension.json`

**Responsibility:** Provide platform-specific plugin loading and hook configuration for each supported AI coding environment.

| Platform | Entry Point | Hook Config | Skills Location |
|----------|-------------|-------------|----------------|
| Claude Code | `plugin.json` + `hooks/session-start` | `hooks/hooks.json` | `skills/` |
| Cursor | `plugin.json` + `hooks/session-start` | `hooks/hooks-cursor.json` | `skills/` via plugin.json |
| OpenCode | `.opencode/plugins/superpowers.js` | Plugin config hook | Dynamically injected |
| Codex | `.codex/INSTALL.md` | Instructions only | Reference docs |
| Gemini CLI | `gemini-extension.json` | Native skill activation | Skills auto-loaded |

**Key Abstraction:**
```javascript
// OpenCode plugin exports (superpowers.js)
export const SuperpowersPlugin = async ({ client, directory }) => ({
  config: async (config) => { /* inject skills path */ },
  'experimental.chat.system.transform': async (input, output) => {
    /* inject bootstrap content via system prompt */
  }
});
```

### 2. Hook System Layer

**Directory:** `hooks/`

**Files:**
- `session-start` - Session initialization script (bash)
- `hooks.json` - Claude Code hook configuration
- `hooks-cursor.json` - Cursor hook configuration
- `run-hook.cmd` - Cross-platform polyglot wrapper (cmd/bash)

**Responsibility:** Initialize skill context at session start by injecting the `using-superpowers` skill content into the agent's context window.

**Communication Flow:**
```
Platform Hook Trigger → run-hook.cmd → session-start script
                                              ↓
                              Read skills/using-superpowers/SKILL.md
                                              ↓
                              Escape for JSON embedding
                                              ↓
                              Output platform-specific format
                                              ↓
                              Agent receives bootstrap context
```

**Platform Detection Logic:**
```bash
if [ -n "${CURSOR_PLUGIN_ROOT:-}" ]; then
  printf '{ "additional_context": "%s" }\n' "$session_context"
elif [ -n "${CLAUDE_PLUGIN_ROOT:-}" ]; then
  printf '{ "hookSpecificOutput": { "hookEventName": "SessionStart", "additionalContext": "%s" }\n}' "$session_context"
else
  printf '{ "additional_context": "%s" }\n' "$session_context"  # fallback
fi
```

### 3. Skills Library Layer (Core Content)

**Directory:** `skills/`

**14 Skills organized by category:**

| Category | Skills | Pattern |
|----------|--------|---------|
| Testing | `test-driven-development/` | Discipline enforcement (rigid) |
| Debugging | `systematic-debugging/`, `verification-before-completion/` | Discipline enforcement (rigid) |
| Collaboration | `brainstorming/`, `writing-plans/`, `executing-plans/`, `dispatching-parallel-agents/`, `requesting-code-review/`, `receiving-code-review/`, `using-git-worktrees/`, `finishing-a-development-branch/`, `subagent-driven-development/` | Flexible patterns |
| Meta | `writing-skills/`, `using-superpowers/` | Reference guides |

**Skill Structure Convention:**
```
skills/
  skill-name/
    SKILL.md              # Required: main skill definition with frontmatter
    supporting-file.*     # Optional: heavy reference, scripts, examples
```

**Frontmatter Schema:**
```yaml
---
name: skill-name-with-hyphens
description: "Use when [specific triggering conditions]"  # Third-person, max 500 chars
---
```

### 4. Subagent System (Subagent-Driven Development)

**Directory:** `skills/subagent-driven-development/`

**Files:**
- `SKILL.md` - Main skill definition
- `implementer-prompt.md` - Template for implementation subagents
- `spec-reviewer-prompt.md` - Template for spec compliance review
- `code-quality-reviewer-prompt.md` - Template for code quality review

**Architecture:** Controller pattern where main agent coordinates specialized subagents:
```
Controller (main agent)
    │
    ├──> Implementer Subagent (Task N)
    │         │
    │         ├──> Spec Reviewer Subagent ──→ [Pass] ──> Code Quality Reviewer
    │         │              ↑
    │         │              └── [Fail] ──> Implementer fixes ──> Re-review
    │         │
    │         └── [Fail] ──> Implementer escalates
    │
    ├──> [Next Task]...
    │
    └──> Final Code Reviewer Subagent
```

**Subagent Communication:** Status reporting via structured format:
- `DONE` - Task completed successfully
- `DONE_WITH_CONCERNS` - Completed but with doubts
- `NEEDS_CONTEXT` - Missing information to proceed
- `BLOCKED` - Cannot complete task

### 5. Visual Brainstorming Subsystem

**Directory:** `skills/brainstorming/scripts/`

**Files:**
- `server.cjs` - WebSocket server (Node.js, zero dependencies)
- `frame-template.html` - HTML frame with CSS styling
- `helper.js` - Client-side interaction helper
- `start-server.sh` / `stop-server.sh` - Platform-specific launch scripts

**Architecture:**
```
Agent writes HTML ──> screen_dir/ ──> server.cjs (file watcher)
                                           │
                                           ├──> HTTP endpoint (GET /)
                                           │         └──> Serves newest HTML wrapped in frame
                                           │
                                           └──> WebSocket endpoint
                                                     └──> Broadcasts reload events

User browser ──> http://localhost:PORT ──> User interactions
                                                     └──> events file (JSONL)
                                                             └──> Agent reads on next turn
```

**Key Design Decisions:**
- **Zero dependencies** - Raw Node.js `http` and `crypto` modules only
- **File-based IPC** - Screen updates via filesystem, events via JSONL
- **Owner process tracking** - Server auto-shuts down if parent process dies
- **30-minute idle timeout** - Prevents orphaned servers

### 6. Agent Prompts Layer

**Directory:** `agents/`, `commands/`

**Files:**
- `agents/code-reviewer.md` - Standalone code review prompt
- `commands/brainstorm.md` - Brainstorm command
- `commands/execute-plan.md` - (Deprecated, redirects to skill)
- `commands/write-plan.md` - Write plan command

**Responsibility:** Standalone prompts for direct invocation without skill activation.

---

## Key Design Patterns

### 1. Frontmatter-Driven Discovery

Skills use YAML frontmatter for discovery by AI agents. The `description` field serves as the triggering condition - agents read this to decide whether to invoke the skill.

```yaml
---
name: test-driven-development
description: Use when implementing any feature or bugfix, before writing implementation code
---
```

### 2. Flowchart-Driven Process Control

Complex skills use Graphviz flowcharts (`digraph`) to define decision trees. This provides:
- Visual process representation
- Decision point clarity
- Loop/iteration handling

Example from `subagent-driven-development`:
```dot
digraph process {
    "Dispatch implementer" -> "Implementer asks questions?";
    "Implementer asks questions?" -> "Answer questions" [label="yes"];
    "Implementer asks questions?" -> "Implement" [label="no"];
    ...
}
```

### 3. Rationalization Tables

Discipline-enforcing skills include explicit tables of rationalizations and responses to prevent agents from circumventing the process:

```markdown
| Excuse | Reality |
|--------|---------|
| "I'll test after" | Tests passing immediately prove nothing. |
| "Just this once" | Delete code. Start over. |
```

### 4. TDD-Applied-to-Documentation

The `writing-skills` skill itself applies TDD methodology:
- **RED:** Run baseline scenario without skill, document rationalizations
- **GREEN:** Write skill addressing those rationalizations
- **REFACTOR:** Close loopholes, re-test until bulletproof

### 5. Cross-Platform Polyglot Hooks

The `run-hook.cmd` uses a polyglot pattern (shell script embedded in cmd batch file):
```batch
: << 'CMDBLOCK'
@echo off
REM Windows: cmd.exe runs batch portion
...
:CMDBLOCK

# Unix: shell interprets this as script
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
exec bash "${SCRIPT_DIR}/${SCRIPT_NAME}" "$@"
```

### 6. Interface Segregation for Skills

Skills reference each other via explicit requirement markers rather than force-loading:
- `**REQUIRED SUB-SKILL:** Use superpowers:test-driven-development`
- NOT `@skills/path/file.md` (which would force-load)

### 7. Two-Stage Review Pattern

The subagent-driven development skill uses a two-stage review after each task:
1. **Spec compliance review** - Did implementation match requirements?
2. **Code quality review** - Is implementation well-built?

This prevents both under-building (missed requirements) and over-building (YAGNI violations).

---

## Dependency Relationships

```
Platform Plugins
    │
    └──> Hook System Layer
              │
              └──> Skills Library (skills/)
                        │
                        ├──> using-superpowers/ (entry point for all skills)
                        │
                        ├──> brainstorming/ ──> visual-companion.md ──> server.cjs
                        │
                        ├──> subagent-driven-development/
                        │         ├──> implementer-prompt.md
                        │         ├──> spec-reviewer-prompt.md
                        │         └──> code-quality-reviewer-prompt.md
                        │
                        ├──> test-driven-development/
                        │         └──> (uses TDD cycle: RED → GREEN → REFACTOR)
                        │
                        ├──> systematic-debugging/
                        │         ├──> root-cause-tracing.md
                        │         ├──> condition-based-waiting.md
                        │         └──> defense-in-depth.md
                        │
                        └──> writing-skills/ ──> testing-skills-with-subagents.md
```

---

## Architectural Decisions with Evidence

### Decision 1: Skills Over Commands

**Decision:** Skills are the primary unit of reusable workflow, not commands or agents.

**Evidence:** Every skill has `name` and `description` in frontmatter optimized for AI discovery. The `using-superpowers` skill explicitly instructs agents to use the `Skill` tool to access other skills. Commands in `commands/` are deprecated or minimal.

### Decision 2: Markdown as Skill Format

**Decision:** Skills are markdown files with YAML frontmatter, not JSON, YAML, or code.

**Evidence:**
- All 14 skills follow the `SKILL.md` naming convention
- Frontmatter uses standard YAML parsing
- Content uses markdown with embedded flowcharts (Graphviz `dot` syntax)
- Zero dependency on skill processing library at runtime

### Decision 3: Flat Namespace for Skills

**Decision:** All skills live in a single `skills/` directory, not nested categories.

**Evidence:**
```bash
skills/
  brainstorming/
  dispatching-parallel-agents/
  executing-plans/
  finishing-a-development-branch/
  receiving-code-review/
  requesting-code-review/
  subagent-driven-development/
  systematic-debugging/
  test-driven-development/
  using-git-worktrees/
  using-superpowers/
  verification-before-completion/
  writing-plans/
  writing-skills/
```

**Rationale:** Simpler discovery, no hierarchy traversal, collision prevention via unique names.

### Decision 4: Zero-Dependency Server

**Decision:** The visual brainstorming server uses only Node.js built-in modules (`http`, `crypto`, `fs`).

**Evidence from `server.cjs`:**
```javascript
const crypto = require('crypto');
const http = require('http');
const fs = require('fs');
const path = require('path');
// No npm dependencies
```

**Rationale:** Portable, no installation issues, works in restricted environments.

### Decision 5: File-Based IPC for Visual Companion

**Decision:** Visual companion uses filesystem (screen files + events file) rather than in-memory state or database.

**Evidence:**
- Screen updates: Write HTML to `screen_dir/`, server watches and serves newest
- User events: Server writes clicks to `state_dir/events` as JSONL
- Agent reads events file on next turn

**Rationale:** Survives server restarts, debuggable, works across process boundaries.

---

## Cross-Cutting Concerns

### Skill Interdependencies

Skills explicitly declare dependencies on other skills:
- `subagent-driven-development` requires `using-git-worktrees`, `writing-plans`, `requesting-code-review`, `finishing-a-development-branch`
- `systematic-debugging` references `test-driven-development` for bug fix verification
- `writing-skills` requires understanding `test-driven-development` first

### Platform Adaptation Layer

Each skill is written for Claude Code tool names. Platform adapters provide tool mappings:
- **OpenCode:** `TodoWrite` → `todowrite`, `Skill` tool → native `skill` tool
- **Gemini CLI:** Tool mapping loaded via `GEMINI.md`
- **Codex:** Reference docs in `references/codex-tools.md`

### Testing Methodology

Skills use subagent-based testing:
- Pressure scenarios with multiple pressure types (time, sunk cost, authority, exhaustion)
- Baseline testing (without skill) to document rationalizations
- Iterative refinement until skill is bulletproof

---

## Summary

Superpowers implements a **skills-based plugin architecture** where:

1. **Platform plugins** adapt to different AI coding environments via platform-specific hook configurations
2. **Hook system** injects skill context at session initialization
3. **Skills library** is a flat namespace of markdown-based workflow templates
4. **Subagent system** enables parallel task execution with two-stage review
5. **Visual brainstorming** provides real-time visual collaboration via zero-dependency WebSocket server
6. **TDD methodology** applies to skill creation itself (write failing test, write skill, close loopholes)

The architecture prioritizes **agent discoverability** (frontmatter-based), **process enforcement** (rationalization tables, red flags), and **zero dependencies** (pure markdown + shell scripts).
