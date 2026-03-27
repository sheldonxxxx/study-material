# Tech Stack

## Overview

The agency-agents repository uses a **minimal, toolchain-agnostic stack** built entirely on plain text and shell scripting. There are no runtime dependencies, no package manager requirements, and no build system. The entire repository consists of Markdown files with YAML frontmatter and Bash scripts for tooling.

## Core Technology

| Component | Technology | Purpose |
|-----------|------------|---------|
| Agent Files | Markdown + YAML | Agent definitions with structured metadata |
| Frontmatter | YAML | Name, description, color, emoji, vibe |
| Scripts | Bash (shell) | Conversion, installation, linting |
| CI/CD | GitHub Actions | Automated validation on PRs |

## Architecture

### Agent File Structure

Each agent is a single Markdown file with two-part structure:

1. **YAML Frontmatter** — Metadata parsed by tools
   ```yaml
   ---
   name: Agent Name
   description: One-line description
   color: colorname or "#hexcode"
   emoji: 🎯
   vibe: One-line personality hook
   services:                              # optional
     - name: Service Name
       url: https://service-url.com
       tier: free
   ---
   ```

2. **Markdown Body** — Human-readable agent definition with sections:
   - Identity & Memory (persona)
   - Communication Style
   - Critical Rules
   - Core Mission
   - Technical Deliverables
   - Workflow Process
   - Success Metrics
   - Advanced Capabilities

### Agent Categories (12 Divisions)

```
academic/          — Scholarly specialists (anthropologist, geographer, historian)
design/            — UX/UI and creative specialists
engineering/      — Software development specialists
game-development/ — Game design and development (Unity, Unreal, Godot, Roblox, Blender)
marketing/         — Growth and marketing specialists (includes China-market specialists)
paid-media/       — Paid acquisition and media specialists
product/           — Product management specialists
project-management/— PM and coordination specialists
sales/             — Sales and revenue specialists
spatial-computing/ — AR/VR/XR specialists
specialized/       — Unique specialists that don't fit elsewhere
testing/           — QA and testing specialists
support/           — Operations and support specialists
```

### Multi-Tool Integration System

The repository ships with a **conversion pipeline** that transforms agent Markdown files into tool-specific formats:

| Tool | Output Format | Integration Path |
|------|--------------|------------------|
| Claude Code | `.md` | `~/.claude/agents/` |
| GitHub Copilot | `.md` | `~/.github/agents/`, `~/.copilot/agents/` |
| Antigravity | `SKILL.md` | `~/.gemini/antigravity/skills/` |
| Gemini CLI | `SKILL.md` + manifest | `~/.gemini/extensions/` |
| OpenCode | `.md` | `.opencode/agents/` |
| Cursor | `.mdc` rule | `.cursor/rules/` |
| Aider | `CONVENTIONS.md` | Project root |
| Windsurf | `.windsurfrules` | Project root |
| OpenClaw | `SOUL.md`, `AGENTS.md`, `IDENTITY.md` | `~/.openclaw/` |
| Qwen Code | `.md` SubAgent | `~/.qwen/agents/` |

### Scripts

| Script | Language | Purpose |
|--------|----------|---------|
| `scripts/convert.sh` | Bash | Converts agent files to all tool formats |
| `scripts/install.sh` | Bash | Installs converted agents to tool-specific paths |
| `scripts/lint-agents.sh` | Bash | Validates frontmatter and structure |

### Linter

The `lint-agents.sh` script enforces:
- **Required frontmatter**: `name`, `description`, `color`
- **Recommended sections**: Identity, Core Mission, Critical Rules
- **Minimum content**: 50 words in body

## Key Design Decisions

1. **No Runtime Dependencies** — The repository is a pure content repository. No Node.js, Python, or other runtime required to use the agents.

2. **Markdown as Source of Truth** — Human-readable source that works without any toolchain while still being machine-parseable.

3. **Shell Scripts Over Build Systems** — Simple, portable Bash scripts that work on any Unix-like system without installing dependencies.

4. **Separation of Concerns** — Conversion (generating integration files) is separate from Installation (placing files in tool-specific locations).

5. **Parallel Processing** — The convert script supports `--parallel` flag for faster multi-tool conversion using `xargs -P`.

6. **Color Normalization** — OpenCode integration includes color name to hex mapping (e.g., `cyan` -> `#00FFFF`).

7. **Section Splitting** — OpenClaw conversion intelligently splits agent files into SOUL.md (persona) vs AGENTS.md (operations) based on section headers.

## Statistics

- **144+ Specialized Agents** across 12 divisions
- **10,000+ lines** of personality, process, and code examples
- **10 supported tools** with conversion scripts
- **Shell-only tooling** — no npm, pip, or other package managers required
