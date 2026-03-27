# Documentation

## Overview

The agency-agents repository maintains comprehensive documentation through a combination of root-level files, in-repo documentation, and community translations. All documentation is Markdown-based with no external documentation generator required.

## Documentation Structure

```
agency-agents/
├── README.md              — Main catalog and quick start
├── CONTRIBUTING.md       — Detailed contribution guidelines
├── LICENSE               — MIT license
├── examples/             — Real-world usage examples
│   ├── README.md
│   ├── nexus-spatial-discovery.md
│   ├── workflow-book-chapter.md
│   ├── workflow-landing-page.md
│   ├── workflow-startup-mvp.md
│   └── workflow-with-memory.md
├── integrations/         — Tool-specific documentation
│   ├── README.md
│   ├── claude-code/README.md
│   ├── github-copilot/README.md
│   ├── antigravity/README.md
│   ├── gemini-cli/README.md
│   ├── opencode/README.md
│   ├── cursor/README.md
│   ├── aider/README.md
│   ├── windsurf/README.md
│   ├── openclaw/README.md
│   └── mcp-memory/README.md
├── scripts/               — Tooling documentation (in script comments)
│   ├── convert.sh
│   ├── install.sh
│   └── lint-agents.sh
└── .github/
    ├── ISSUE_TEMPLATE/   — Structured issue forms
    │   ├── bug-report.yml
    │   └── new-agent-request.yml
    └── PULL_REQUEST_TEMPLATE.md
```

## Key Documentation Files

### README.md

The main entry point containing:

- **What Is This** — Project overview and philosophy
- **Quick Start** — Three integration paths (Claude Code, reference, multi-tool)
- **The Agency Roster** — Complete agent catalog organized by division
- **Real-World Use Cases** — Example team configurations
- **Contributing** — Brief contribution guide
- **Agent Design Philosophy** — Core principles
- **Multi-Tool Integrations** — Detailed setup for all 10 supported tools
- **Roadmap** — Future plans
- **Community Translations** — Links to community-maintained translations
- **Related Resources** — External resources
- **Stats** — Repository metrics

### CONTRIBUTING.md

Comprehensive contribution guide including:

1. **Code of Conduct** — Community guidelines
2. **How Can I Contribute** — Four contribution paths
3. **Agent Design Guidelines** — Full agent template and structure
4. **Pull Request Process** — What belongs in a PR, review workflow
5. **Style Guide** — Writing style, formatting, tone
6. **Recognition** — Contributor acknowledgment

### Agent Template

The CONTRIBUTING.md defines the canonical agent structure:

```markdown
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

# Agent Name

## Your Identity & Memory
- Role: Clear role description
- Personality: Communication style
- Memory: What the agent remembers
- Experience: Domain expertise

## Your Core Mission
- Primary responsibility 1
- Primary responsibility 2
- Default requirement: Always-on best practices

## Critical Rules You Must Follow
Domain-specific rules and constraints

## Your Technical Deliverables
Code samples, templates, frameworks

## Your Workflow Process
Step-by-step methodology

## Your Communication Style
Tone, voice, example phrases

## Learning & Memory
Pattern recognition, improvement

## Your Success Metrics
Measurable outcomes, benchmarks

## Advanced Capabilities
Specialized techniques
```

### Examples Directory

Real-world usage examples demonstrating agent team compositions:

| Example | Description |
|---------|-------------|
| `nexus-spatial-discovery.md` | 8 agents deployed simultaneously for product opportunity evaluation |
| `workflow-startup-mvp.md` | Team assembly for rapid startup development |
| `workflow-landing-page.md` | Marketing + design agent collaboration |
| `workflow-book-chapter.md` | Writing workflow with specialized agents |
| `workflow-with-memory.md` | Multi-agent coordination with memory systems |

## Integration Documentation

Each integration has a dedicated README explaining:

- Installation method
- Activation syntax
- Usage patterns
- Tool-specific considerations

## Issue and PR Templates

### Bug Report (`.github/ISSUE_TEMPLATE/bug-report.yml`)
- Structured form with fields for agent name, category, specialty
- Checklist for verification steps
- Expected vs actual behavior

### New Agent Request (`.github/ISSUE_TEMPLATE/new-agent-request.yml`)
- Agent name, category, specialty
- Motivation and use case
- Existing alternatives

### PR Template (`.github/PULL_REQUEST_TEMPLATE.md`)
- Agent information section
- Motivation
- Testing evidence
- Checklist (template structure, personality, examples, metrics, workflow)

## Agent Design Principles

Documented in CONTRIBUTING.md:

1. **Strong Personality** — Distinct voice, not generic
2. **Clear Deliverables** — Concrete code examples, not vague guidance
3. **Success Metrics** — Specific, measurable outcomes
4. **Proven Workflows** — Step-by-step, battle-tested processes
5. **Learning Memory** — Pattern recognition, continuous improvement

## External Services Documentation

Agents may declare external service dependencies in frontmatter:

```yaml
services:
  - name: Service Name
    url: https://service-url.com
    tier: free  # free, freemium, or paid
```

Guidelines:
- Declare dependencies in frontmatter
- Agent must stand alone without API calls
- Don't duplicate vendor docs
- Prefer free-tier services for testing

## Community Translations

| Language | Maintainer | Link | Notes |
|----------|-----------|------|-------|
| zh-CN | @jnMetaCode | agency-agents-zh | 100 translated + 9 China originals |
| zh-CN | @dsclca12 | agent-teams | Independent with Bilibili/WeChat/Xiaohongshu |

## Documentation Quality Standards

- **Specificity** — "Reduce page load by 60%" not "Make it faster"
- **Concreteness** — "Create React components with TypeScript" not "Build UIs"
- **Memorability** — Personality-driven, not generic corporate speak
- **Practicality** — Real code examples, not pseudo-code
- **Professional tone** — Confident but not arrogant, helpful but not hand-holding
