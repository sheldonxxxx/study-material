# Project Overview: agency-agents

## What Is This Project?

**agency-agents** (also known as "The Agency") is a curated collection of 144+ AI agent personalities designed for use with AI coding tools. Each agent is a specialized expert with a distinct personality, defined workflows, and measurable deliverables — not generic prompt templates.

The repository provides agents across 12 functional divisions covering engineering, design, marketing, sales, game development, testing, and specialized domains. Agents are distributed as markdown files that integrate natively with Claude Code and are converted via scripts for GitHub Copilot, Cursor, Windsurf, Aider, Gemini CLI, and other AI coding tools.

## Project Scope

### Core Value Proposition

- **Specialization**: Deep expertise in specific domains rather than general-purpose assistance
- **Personality**: Each agent has a distinct voice, communication style, and approach
- **Deliverables**: Concrete outputs — code examples, documents, strategies — not vague guidance
- **Production-Ready**: Battle-tested workflows with success metrics and quality standards

### Agent Divisions

| Division | Agents | Focus Areas |
|----------|--------|-------------|
| Engineering | 25 | Frontend, backend, mobile, DevOps, security, data engineering |
| Design | 8 | UI design, UX research, brand, visual storytelling |
| Marketing | 29 | Content, social media, SEO, paid media, China platforms |
| Sales | 8 | Outbound, discovery, deal strategy, proposals |
| Game Development | 23 | Unity, Unreal, Godot, Roblox, Blender, cross-engine |
| Testing | 8 | Evidence collection, reality checking, accessibility |
| Specialized | 29 | Compliance, blockchain, document generation, MCP builder |
| Other Divisions | 14 | Paid media (7), Academic (5), Spatial computing (6), Support (5), Product (5), Project management (6) |

### Integration Ecosystem

The project ships with conversion and installation scripts supporting 10+ AI coding tools:

- **Claude Code** (primary) — agents copied directly to `~/.claude/agents/`
- **GitHub Copilot** — agents in `~/.github/agents/` and `~/.copilot/agents/`
- **Cursor** — converted to `.mdc` rule files in `.cursor/rules/`
- **Windsurf** — compiled into `.windsurfrules`
- **Aider** — consolidated into single `CONVENTIONS.md`
- **Gemini CLI** — extension format with per-agent SKILL.md files
- **Antigravity** — skill format for Google Gemini
- **OpenCode**, **OpenClaw**, **Qwen Code** — various agent file formats

### Repository Structure

```
agency-agents/
  engineering/          # 25 software engineering agents
  design/              # 8 design specialists
  game-development/     # 23 game dev agents (Unity, Unreal, Godot, etc.)
  marketing/           # 29 marketing specialists
  paid-media/          # 7 paid acquisition agents
  product/             # 5 product management agents
  project-management/  # 6 PM/coordinator agents
  testing/             # 8 QA/testing agents
  support/             # 5 operations/support agents
  spatial-computing/    # 6 AR/VR/XR agents
  specialized/         # 29 domain-specific agents
  academic/            # 5 academic specialists
  integrations/        # Tool-specific conversion documentation
  scripts/
    convert.sh         # Generates integration files for all tools
    install.sh         # Installs agents to tool-specific locations
    lint-agents.sh      # Validates agent markdown files
  examples/            # Multi-agent workflow examples
```

## Agent File Format

Each agent is a markdown file with YAML frontmatter:

```yaml
---
name: Agent Name
description: One-line description of specialty and focus
color: colorname or "#hexcode"
emoji: 🎯
vibe: One-line personality hook
---
```

The body follows a structured template:

1. **Identity & Memory** — Role, personality, memory patterns, experience
2. **Core Mission** — Primary responsibilities and workflows
3. **Critical Rules** — Domain-specific constraints and principles
4. **Technical Deliverables** — Code examples, documents, templates
5. **Workflow Process** — Step-by-step operational methodology
6. **Communication Style** — How the agent communicates
7. **Learning & Memory** — Pattern recognition and expertise building
8. **Success Metrics** — Measurable outcomes

## Key Statistics

- **147 agents** across 12 divisions
- **10,000+ lines** of personality, process, and code examples
- **10+ tool integrations** via conversion scripts
- **MIT License** — free for commercial and personal use
- **Community-driven** — submissions via GitHub PRs

## Usage Patterns

### Single Agent Activation
```bash
# Copy agents to Claude Code
cp -r agency-agents/* ~/.claude/agents/

# Activate in conversation
"Use the Frontend Developer agent to build this component"
```

### Multi-Agent Workflows
The project includes examples of coordinated multi-agent sessions where agents from different divisions work together on complex projects (e.g., startup MVP building, product discovery exercises).

## Quality Infrastructure

The project includes:
- **lint-agents.sh** — Validates frontmatter structure and content
- **Testing Division** — Dedicated QA agents (Evidence Collector, Reality Checker) that require visual proof
- **Code Reviewer Agent** — Mentorship-focused review with blocker/suggestion/nit classification
- **Security Engineer Agent** — Threat modeling, secure code review, security architecture

## Security Approach

The Security Engineer agent defines:
- Threat modeling methodology (STRIDE analysis)
- Secure coding patterns (input validation, parameterized queries)
- Security headers configuration
- CI/CD security scanning pipeline (Semgrep, Trivy, Gitleaks)
- Zero-trust architecture principles

## Project Health Indicators

- Active GitHub community (discussions, PRs)
- Multiple community translations (Chinese localization)
- Regular updates and new agent additions
- Comprehensive CONTRIBUTING.md with agent design guidelines
