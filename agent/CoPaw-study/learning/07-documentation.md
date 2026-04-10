# CoPaw Documentation

## Overview

CoPaw maintains comprehensive documentation with **bilingual support** (English + Chinese) and multiple distribution formats.

## Documentation Locations

| Location | Format | URL |
|----------|--------|-----|
| **Website** | React/Vite | https://copaw.agentscope.io/ |
| **Repo docs** | Static Markdown | `website/public/docs/` |
| **README** | Markdown | `README.md`, `README_zh.md`, `README_ja.md` |
| **Contributing** | Markdown | `CONTRIBUTING.md`, `CONTRIBUTING_zh.md` |
| **Release Notes** | Markdown | `website/public/release-notes/` |
| **Agent Prompts** | Markdown | `src/copaw/agents/md_files/{lang}/` |

## Website Document Structure

All docs in `website/public/docs/` have **English (`.en.md`)** and **Chinese (`.zh.md`)** versions:

```
website/public/docs/
├── channels.en.md / .zh.md      # Messaging channel setup
├── cli.en.md / .zh.md           # CLI reference
├── commands.en.md / .zh.md      # Magic commands
├── community.en.md / .zh.md     # Community resources
├── comparison.en.md / .zh.md    # vs other assistants
├── config.en.md / .zh.md        # Configuration
├── console.en.md / .zh.md      # Console UI guide
├── contributing.en.md / .zh.md  # Contribution guide
├── context.en.md / .zh.md       # Context management
├── desktop.en.md / .zh.md       # Desktop app guide
├── faq.en.md / .zh.md           # FAQ
├── heartbeat.en.md / .zh.md     # Scheduled tasks
├── intro.en.md / .zh.md         # Introduction
├── mcp.en.md / .zh.md           # MCP support
├── memory.en.md / .zh.md        # Memory system
├── models.en.md / .zh.md        # AI model providers
├── multi-agent.en.md / .zh.md   # Multi-agent setup
├── quickstart.en.md / .zh.md   # Quick start guide
├── roadmap.en.md / .zh.md      # Project roadmap
├── security.en.md / .zh.md      # Security policy
├── skills.en.md / .zh.md        # Skills system
```

## Agent Prompt Files

Agent behavior defined by Markdown files in `src/copaw/agents/md_files/`:

```
src/copaw/agents/md_files/
├── {lang}/
│   ├── AGENTS.md      # Agent configuration
│   ├── BOOTSTRAP.md   # Initialization
│   ├── HEARTBEAT.md   # Heartbeat behavior
│   ├── MEMORY.md      # Memory handling
│   ├── PROFILE.md     # User profile
│   └── SOUL.md       # Core personality
├── qa/
│   └── (QA agent prompts)
```

**Supported Languages:** `en` (English), `zh` (Chinese), `ru` (Russian)

## Skill Documentation

Skills have their own documentation within `src/copaw/agents/skills/`:

```
skills/
├── SKILL.md           # Main skill definition
├── references/        # Reference materials
├── scripts/          # Helper scripts
├── pdf/
│   ├── SKILL.md
│   ├── forms.md
│   └── reference.md
├── pptx/
│   ├── SKILL.md
│   ├── editing.md
│   └── pptxgenjs.md
└── himalaya/
    ├── SKILL.md
    └── references/configuration.md
```

## What Is Well Documented

### 1. Getting Started
- Quick start guide (pip, script install, Docker, ModelScope)
- Desktop application installation
- API key configuration
- Local model setup (llama.cpp, MLX, Ollama)

### 2. Core Concepts
- Channels (DingTalk, Feishu, QQ, Discord, iMessage, etc.)
- Skills system and management
- Memory and context
- Heartbeat (scheduled tasks)
- Multi-agent support

### 3. Configuration
- Working directory structure
- Config file format
- Environment variables
- CLI commands

### 4. Development
- Contributing guidelines (English + Chinese)
- Conventional commits format
- Adding new channels, models, skills
- Platform support (Windows, Linux, macOS)

### 5. Security
- Security policy with ASRC integration
- Private disclosure process
- Trusted Skills concept
- File access guards

### 6. Release Notes
- Comprehensive changelog for each version
- Bilingual release notes

## Documentation Gaps

### 1. Architecture Documentation
- No dedicated architecture doc
- No sequence diagrams for channel message flow
- No detailed component relationship diagrams

### 2. API Reference
- No generated API documentation (e.g., Sphinx, OpenAPI)
- REST API endpoints not formally documented
- Type hints exist but not extracted to docs

### 3. Code Examples
- Limited code examples in docs
- No interactive tutorials
- Skills examples could be more comprehensive

### 4. Troubleshooting
- FAQ covers basics but not deep debugging
- No error code reference
- No debug logging guide

### 5. Deployment
- Basic Docker instructions
- No Kubernetes/helm charts
- No production deployment best practices
- No monitoring/observability docs

### 6. Internal Implementation
- No internals documentation
- How channels integrate with agents
- How skills are loaded and executed
- Provider registry mechanism

## Contributing to Documentation

Per CONTRIBUTING.md:

```bash
# Edit docs (English primary)
website/public/docs/*.en.md

# Edit Chinese translation
website/public/docs/*.zh.md

# Edit agent prompts
src/copaw/agents/md_files/{lang}/

# Edit skill docs
src/copaw/agents/skills/*/SKILL.md
```

Documentation updates should accompany user-facing code changes.

## Doc Build and Deploy

Website uses React + Vite. Docs are static Markdown files served by the React app. Deployment is automatic via `deploy-website.yml` on push to main.
