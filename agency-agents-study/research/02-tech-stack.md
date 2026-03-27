# Tech Stack Analysis: agency-agents

## Overview

**Repository Type:** Documentation/Agent Definition Collection
**Primary Language:** Markdown
**Build System:** None (pure documentation)
**CI/CD:** GitHub Actions (markdown linting only)

This repository is not a traditional software project. It is a curated collection of AI agent personality definitions written in Markdown format, designed to be consumed by various AI coding tools (Claude Code, Cursor, GitHub Copilot, etc.).

---

## 1. Dependencies Analysis

### 1.1 Traditional Package Managers

| Manager | Found | Files |
|---------|-------|-------|
| npm/Node | No | - |
| Go | No | - |
| Rust/Cargo | No | - |
| Python/pip | No | - |
| Ruby/Bundler | No | - |

**Finding:** No traditional language-specific dependency files exist. The repository contains only Markdown files.

### 1.2 Dependency Declaration

The repository has **no external code dependencies**. Agent definitions are self-contained Markdown documents with YAML frontmatter.

---

## 2. Languages & Frameworks

### 2.1 Primary Language

| Language | Files | Purpose |
|----------|-------|---------|
| **Markdown** | 144+ .md files | Agent personality definitions |

### 2.2 Secondary Languages (in code examples within agents)

The agent definitions contain code examples demonstrating deliverables. These are illustrative only:

| Language | Occurrence | Purpose |
|----------|-----------|---------|
| TypeScript/TSX | Common | React component examples |
| JavaScript | Common | General code examples |
| Bash/Shell | In scripts/ | Build/conversion scripts |
| Python | Rare | Data pipeline examples |
| SQL | Rare | Database examples |
| Solidity | Rare | Smart contract examples |
| GDScript | Rare | Godot game examples |
| Luau | Rare | Roblox examples |

### 2.3 Frameworks Referenced (in agent examples, not project deps)

| Framework | Context |
|-----------|---------|
| React | Frontend Developer agent |
| Vue/Angular/Svelte | Frontend Developer agent |
| Tailwind CSS | Styling examples |
| Next.js | Frontend examples |
| Node.js/Express | Backend examples |
| Laravel | Backend Architect |
| PostgreSQL/MySQL | Database Optimizer |
| Unity | Game Development division |
| Unreal Engine | Game Development division |
| Godot | Game Development division |
| Blender (bpy) | Game Development division |
| Roblox Studio (Luau) | Game Development division |

---

## 3. Build System

### 3.1 Build Tools

| Tool | Present | Purpose |
|------|---------|---------|
| Make | No | - |
| CMake | No | - |
| Webpack | No | - |
| Vite | No | - |
| Turborepo | No | - |
| Nx | No | - |
| Lerna | No | - |

**Finding:** No build system. The repository is deployed as-is.

### 3.2 Scripts Directory

Location: `/scripts/`

| Script | Purpose |
|--------|---------|
| `convert.sh` | Converts agent .md files to tool-specific formats |
| `install.sh` | Installs converted agents to tool-specific directories |
| `lint-agents.sh` | Validates frontmatter and structure of agent files |

These are bash scripts for **agent file transformation**, not a build system for compiling code.

---

## 4. Runtime & Environment

### 4.1 Runtime Environments

| Runtime | Present |
|---------|---------|
| Node.js | No (not required) |
| Python | No (not required) |
| Go | No (not required) |
| Docker | No |

### 4.2 Execution Model

Agents are **consumed at runtime by AI tools**, not executed as server-side code. The markdown files are loaded by:
- Claude Code (direct copy to `~/.claude/agents/`)
- Cursor (converted to `.cursor/rules/*.mdc` files)
- GitHub Copilot (direct copy to `~/.github/agents/`)
- Aider (compiled into `CONVENTIONS.md`)
- Windsurf (compiled into `.windsurfrules`)
- Other tools via `convert.sh` transformations

---

## 5. CI/CD Configuration

### 5.1 GitHub Actions

Location: `/github/workflows/`

**File:** `lint-agents.yml`

```yaml
name: Lint Agent Files
on:
  pull_request:
    paths:
      - 'design/**'
      - 'engineering/**'
      - 'game-development/**'
      - 'marketing/**'
      - 'paid-media/**'
      - 'sales/**'
      - 'product/**'
      - 'project-management/**'
      - 'testing/**'
      - 'support/**'
      - 'spatial-computing/**'
      - 'specialized/**'

jobs:
  lint:
    name: Validate agent frontmatter and structure
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run agent linter
        run: ./scripts/lint-agents.sh $CHANGED_FILES
```

**Purpose:** Validates YAML frontmatter in agent files (name, description, color required).

### 5.2 Other GitHub Configurations

| File | Purpose |
|------|---------|
| `FUNDING.yml` | GitHub Sponsors integration |
| `ISSUE_TEMPLATE/bug-report.yml` | Bug report form |
| `ISSUE_TEMPLATE/new-agent-request.yml` | Agent request form |
| `PULL_REQUEST_TEMPLATE.md` | PR description template |

---

## 6. Monorepo Structure

### 6.1 Organization

The repository is organized by **domain divisions** (not a technical monorepo):

```
agency-agents/
  academic/          # 5 agents - scholarly/narrative specialists
  design/            # 8 agents - UI/UX/brand specialists
  engineering/       # 23 agents - software development
  game-development/  # 20+ agents - Unity, Unreal, Godot, Roblox, Blender
  integrations/      # Tool-specific output (generated)
  marketing/         # 40+ agents - social, SEO, content
  paid-media/        # 7 agents - advertising
  product/           # 5 agents - product management
  project-management/ # 7 agents - project coordination
  sales/             # 8 agents - sales strategy
  specialized/       # 29 agents - niche specialists
  spatial-computing/ # 6 agents - AR/VR/XR
  support/           # 6 agents - customer support
  testing/           # 8 agents - QA specialists
  scripts/           # Conversion/installation tools
  examples/          # Multi-agent workflow examples
```

### 6.2 Monorepo Tooling

| Tool | Present |
|------|---------|
| Nx | No |
| Turborepo | No |
| Lerna | No |
| Bazel | No |
| Buck | No |

**Finding:** Not a technical monorepo. No shared code or packages. Each agent is independent.

---

## 7. Containerization

### 7.1 Docker

| File | Present |
|------|---------|
| Dockerfile | No |
| docker-compose.yml | No |
| .dockerignore | No |

**Finding:** No containerization. This is not a running service.

---

## 8. Agent File Structure

Each agent follows a consistent Markdown structure:

### 8.1 Frontmatter (YAML)

```yaml
---
name: Agent Name
description: Brief description
color: cyan
emoji: 🖥️
vibe: One-line personality summary
---
```

### 8.2 Body Sections

| Section | Purpose |
|---------|---------|
| Identity & Memory | Role, personality, memory |
| Core Mission | Primary responsibilities |
| Critical Rules | Non-negotiable behaviors |
| Technical Deliverables | Code examples, outputs |
| Workflow Process | Step-by-step methodology |
| Success Metrics | Measurable outcomes |
| Communication Style | How the agent communicates |

---

## 9. Integration System

### 9.1 Supported AI Tools

The repository converts agents to format for:

| Tool | Format | Directory |
|------|--------|-----------|
| Claude Code | `.md` | `~/.claude/agents/` |
| GitHub Copilot | `.md` | `~/.github/agents/` |
| Antigravity | `SKILL.md` | `~/.gemini/antigravity/skills/` |
| Gemini CLI | Extension | `~/.gemini/extensions/` |
| OpenCode | `.md` | `.opencode/agents/` |
| Cursor | `.mdc` | `.cursor/rules/` |
| Aider | `CONVENTIONS.md` | `./CONVENTIONS.md` |
| Windsurf | `.windsurfrules` | `./.windsurfrules` |
| OpenClaw | `SOUL.md/AGENTS.md/IDENTITY.md` | `~/.openclaw/` |
| Qwen Code | `.md` | `~/.qwen/agents/` |

### 9.2 Conversion Pipeline

```
1. convert.sh — Transforms .md agents to tool-specific formats
2. install.sh — Copies/symlinks converted files to tool directories
```

---

## 10. Key Findings Summary

| Aspect | Finding |
|--------|---------|
| **Project Type** | Documentation/Agent collection |
| **Languages** | Markdown (primary), various in examples |
| **Dependencies** | None (no external code deps) |
| **Build System** | None |
| **Runtime** | Consumed by AI tools, not executed |
| **CI/CD** | GitHub Actions (markdown linting only) |
| **Monorepo** | No (categorical organization only) |
| **Containerization** | None |
| **Package Management** | None |
| **Test Framework** | None |

---

## 11. Implications for Development

### 11.1 For Contributing Agents

1. **No code compilation** - Submit markdown files
2. **Frontmatter required** - Must include `name`, `description`, `color`
3. **Lint check** - PRs validated by `lint-agents.sh`
4. **No environment setup** - Pure documentation contribution

### 11.2 For Tooling This Repository

1. **No npm install** - No package.json to parse
2. **No build step** - Files deployed as-is
3. **Shell scripts** - For file transformation only
4. **YAML frontmatter** - Primary structured data

### 11.3 For Running Agents Locally

1. **Copy files** - `cp -r agency-agents/* ~/.claude/agents/`
2. **Or convert** - Run `./scripts/convert.sh && ./scripts/install.sh`
3. **No server** - Agents run within AI tool context
