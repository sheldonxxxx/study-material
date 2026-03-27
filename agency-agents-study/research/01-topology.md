# agency-agents Repository Topology

## Overview

**Repository Type:** AI Agent Definition Knowledge Base
**Total Files:** 193 markdown files (agent definitions)
**Purpose:** Collection of specialized AI agent personalities for various domains (engineering, marketing, design, etc.) with conversion scripts for multiple AI coding tools.

---

## Directory Structure

```
agency-agents/
├── README.md                    # Primary entry point / documentation
├── CONTRIBUTING.md              # Contribution guidelines
├── LICENSE                      # MIT License
├── .gitignore
├── .github/
│   └── workflows/
│       └── lint-agents.yml      # CI linting workflow
│   └── ISSUE_TEMPLATE/
├── scripts/
│   ├── convert.sh               # Converts agents to tool-specific formats
│   ├── install.sh               # Installs agents for specific tools
│   └── lint-agents.sh           # Lints agent markdown files
│
├── engineering/                 # 25 engineering agent definitions
│   ├── engineering-frontend-developer.md
│   ├── engineering-backend-architect.md
│   ├── engineering-mobile-app-builder.md
│   ├── engineering-ai-engineer.md
│   ├── engineering-devops-automator.md
│   ├── engineering-security-engineer.md
│   ├── engineering-solidity-smart-contract-engineer.md
│   ├── ... (25+ total)
│
├── marketing/                  # 29 marketing agent definitions
│   ├── marketing-seo-specialist.md
│   ├── marketing-social-media-strategist.md
│   ├── marketing-douyin-strategist.md
│   ├── marketing-wechat-official-account.md
│   ├── ... (29+ total)
│
├── design/                      # 8 design agent definitions
│   ├── design-ui-designer.md
│   ├── design-ux-architect.md
│   ├── design-ux-researcher.md
│   ├── design-brand-guardian.md
│   ├── ... (8 total)
│
├── specialized/                 # 27+ specialized agent definitions
│   ├── specialized-mcp-builder.md
│   ├── specialized-model-qa.md
│   ├── specialized-workflow-architect.md
│   ├── specialized-salesforce-architect.md
│   ├── ... (27+ total)
│
├── integrations/                 # Tool-specific integration configs
│   ├── aider/
│   ├── antigravity/
│   ├── claude-code/
│   ├── cursor/
│   ├── gemini-cli/
│   ├── github-copilot/
│   ├── mcp-memory/
│   ├── openclaw/
│   ├── opencode/
│   └── windsurf/
│
├── game-development/            # Game dev domain agents
│   ├── unreal-engine/
│   ├── unity/
│   ├── roblox-studio/
│   ├── blender/
│   └── godot/
│
├── examples/                    # Workflow examples
│   ├── nexus-spatial-discovery.md
│   ├── workflow-book-chapter.md
│   ├── workflow-landing-page.md
│   ├── workflow-startup-mvp.md
│   └── workflow-with-memory.md
│
├── academic/                    # Academic domain agents
├── product/                     # Product domain agents
├── sales/                       # Sales domain agents
├── strategy/                    # Strategy domain agents
│   ├── coordination/
│   ├── playbooks/
│   └── runbooks/
├── support/                     # Support domain agents
├── testing/                     # Testing domain agents
├── paid-media/                  # Paid media domain agents
├── project-management/          # PM domain agents
├── spatial-computing/           # Spatial computing domain agents
```

---

## Entry Points

### Primary Entry Points

| File | Purpose |
|------|---------|
| `README.md` | Main documentation, agent roster, quick start guides |
| `scripts/convert.sh` | Converts markdown agents to tool-specific formats (Claude Code, Cursor, Aider, etc.) |
| `scripts/install.sh` | Interactive installer for agents across supported tools |
| `scripts/lint-agents.sh` | Validates agent markdown files |

### Agent Definition Structure

Each agent is a markdown file (`.md`) containing:
- Identity and personality traits
- Core mission and workflows
- Technical deliverables with code examples
- Success metrics
- Communication style

### Tool Integration Entry Points

| Directory | Purpose |
|-----------|---------|
| `integrations/claude-code/` | Claude Code agent configurations |
| `integrations/cursor/` | Cursor IDE agent configurations |
| `integrations/aider/` | Aider CLI agent configurations |
| `integrations/windsurf/` | Windsurf IDE agent configurations |
| `integrations/github-copilot/` | GitHub Copilot agent configurations |
| `integrations/mcp-memory/` | MCP Memory tool configurations |
| `integrations/gemini-cli/` | Google Gemini CLI configurations |
| `integrations/openclaw/` | OpenClaw tool configurations |
| `integrations/opencode/` | OpenCode tool configurations |
| `integrations/antigravity/` | Antigravity tool configurations |

---

## Layout Patterns

### Domain-Based Organization

Agents are organized by business domain rather than technical function:
- `engineering/` - Software development roles
- `marketing/` - Marketing and growth roles
- `design/` - UI/UX and visual design roles
- `specialized/` - Niche/specialized roles
- `game-development/` - Game development (further subdivided by engine)

### Tool-Specific Subdirectories

The `integrations/` directory contains subdirectories for each supported AI coding tool. These are populated by `convert.sh`.

### Workflow Examples

The `examples/` directory contains workflow demonstrations showing how agents collaborate.

---

## Key Files Summary

| Path | Lines | Description |
|------|-------|-------------|
| `README.md` | 51,365 bytes | Main documentation with agent roster |
| `scripts/install.sh` | 21,973 bytes | Interactive installer |
| `scripts/convert.sh` | 17,382 bytes | Format converter |
| `scripts/lint-agents.sh` | 2,645 bytes | Agent linter |
| `CONTRIBUTING.md` | 14,228 bytes | Contribution guidelines |

---

## CI/CD Entry Points

- `.github/workflows/lint-agents.yml` - GitHub Actions workflow that runs `scripts/lint-agents.sh` on pull requests
