# Documentation - Hermes Agent

## Documentation Site

**URL**: [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs)

**Stack**: Docusaurus 3.9.2 with TypeScript

### Site Structure

```
website/
├── docs/
│   ├── index.md                    # Landing page
│   ├── developer-guide/
│   │   ├── _category_.json
│   │   ├── architecture.md
│   │   ├── agent-loop.md
│   │   ├── contributing.md
│   │   ├── creating-skills.md
│   │   ├── adding-tools.md
│   │   ├── adding-providers.md
│   │   ├── prompt-assembly.md
│   │   ├── provider-runtime.md
│   │   ├── tools-runtime.md
│   │   ├── session-storage.md
│   │   ├── context-compression-and-caching.md
│   │   ├── cron-internals.md
│   │   ├── gateway-internals.md
│   │   ├── environments.md
│   │   ├── trajectory-format.md
│   │   └── extending-the-cli.md
│   ├── developer-guide/acp-internals.md
│   ├── getting-started/
│   │   ├── _category_.json
│   │   ├── quickstart.md
│   │   ├── installation.md
│   │   ├── updating.md
│   │   ├── nix-setup.md
│   │   └── learning-path.md
│   ├── guides/
│   │   ├── _category_.json
│   │   ├── daily-briefing-bot.md
│   │   ├── team-telegram-assistant.md
│   │   ├── use-mcp-with-hermes.md
│   │   ├── use-soul-with-hermes.md
│   │   ├── python-library.md
│   │   ├── build-a-hermes-plugin.md
│   │   ├── tips.md
│   │   └── use-voice-mode-with-hermes.md
│   ├── reference/
│   │   ├── _category_.json
│   │   ├── cli-commands.md
│   │   ├── environment-variables.md
│   │   ├── toolsets-reference.md
│   │   ├── skills-catalog.md
│   │   ├── optional-skills-catalog.md
│   │   ├── slash-commands.md
│   │   ├── mcp-config-reference.md
│   │   ├── tools-reference.md
│   │   └── faq.md
│   └── user-guide/
│       ├── _category_.json
│       ├── cli.md
│       ├── configuration.md
│       ├── sessions.md
│       ├── security.md
│       ├── git-worktrees.md
│       ├── checkpoints-and-rollback.md
│       ├── features/
│       │   ├── _category_.json
│       │   ├── tools.md
│       │   ├── skills.md
│       │   ├── memory.md
│       │   ├── personality.md
│       │   ├── cron.md
│       │   ├── mcp.md
│       │   ├── delegation.md
│       │   ├── context-files.md
│       │   ├── context-references.md
│       │   ├── browser.md
│       │   ├── voice-mode.md
│       │   ├── tts.md
│       │   ├── vision.md
│       │   ├── image-generation.md
│       │   ├── hooks.md
│       │   ├── plugins.md
│       │   ├── api-server.md
│       │   ├── code-execution.md
│       │   ├── batch-processing.md
│       │   ├── checkpoints.md
│       │   ├── honcho.md
│       │   ├── rl-training.md
│       │   └── fallback-providers.md
│       └── messaging/
│           ├── _category_.json
│           ├── index.md
│           ├── telegram.md
│           ├── discord.md
│           ├── slack.md
│           ├── whatsapp.md
│           ├── signal.md
│           ├── email.md
│           ├── sms.md
│           ├── matrix.md
│           ├── mattermost.md
│           ├── homeassistant.md
│           ├── dingtalk.md
│           ├── open-webui.md
│           └── webhooks.md
├── package.json
├── docusaurus.config.js
└── sidebars.js
```

## Documentation Sections

### Getting Started
| Page | Purpose |
|------|---------|
| Quickstart | Install and first conversation in 2 minutes |
| Installation | Detailed install instructions for all platforms |
| Updating | How to update to latest version |
| Nix Setup | Nix-based installation |
| Learning Path | Find docs by experience level |

### User Guide
| Section | Topics |
|---------|--------|
| CLI | Commands, keybindings, personalities, sessions |
| Configuration | Config file, providers, models, all options |
| Messaging | Telegram, Discord, Slack, WhatsApp, Signal, Email, SMS, Matrix, DingTalk |
| Security | Command approval, DM pairing, container isolation |
| Features | Tools, Skills, Memory, Personality, Cron, MCP, Delegation, Context Files, Browser, Voice Mode, TTS, Vision, Image Generation, Hooks, Plugins, API Server, Code Execution, Batch Processing, Checkpoints, Honcho, RL Training |
| Sessions | Session management and history |

### Developer Guide
| Page | Purpose |
|------|---------|
| Architecture | Project structure, agent loop, key classes |
| Contributing | Development setup, PR process, code style |
| Creating Skills | SKILL.md format, bundled vs optional skills |
| Adding Tools | Tool registration pattern |
| Adding Providers | Provider implementation |
| Agent Loop | Core conversation loop internals |
| Prompt Assembly | System prompt building |
| Provider Runtime | API call handling |
| Tools Runtime | Tool execution |
| Session Storage | SQLite + FTS5 |
| Context Compression | Token limit handling |
| Cron Internals | Scheduling system |
| Gateway Internals | Messaging gateway architecture |
| Environments | Terminal backends |
| Trajectory Format | Data export format |
| ACP Internals | Agent Client Protocol |
| Extending the CLI | CLI customization |

### Reference
| Reference | Content |
|-----------|---------|
| CLI Commands | All commands and flags |
| Environment Variables | Complete env var reference |
| Toolsets Reference | Tool groupings |
| Skills Catalog | Bundled skills |
| Optional Skills Catalog | Optional skills |
| Slash Commands | Slash command reference |
| MCP Config Reference | MCP configuration |
| Tools Reference | Tool schemas |
| FAQ | Common questions |

### Guides
| Guide | Description |
|-------|-------------|
| Daily Briefing Bot | Automated daily reports |
| Team Telegram Assistant | Multi-user Telegram setup |
| Use MCP with Hermes | MCP integration patterns |
| Use Soul with Hermes | Soul user modeling integration |
| Python Library | Programmatic usage |
| Build a Hermes Plugin | Plugin development |
| Tips | Best practices |
| Use Voice Mode | Voice interaction setup |

## In-Repo Documentation

### Root Files
- `README.md` — Quick install, getting started, feature overview
- `CONTRIBUTING.md` — Development guide, architecture, PR process, code style
- `LICENSE` — MIT license

### Documentation Docs
- `docs/acp-setup.md` — ACP setup for VS Code, Zed, JetBrains
- `docs/honcho-integration-spec.md` — Honcho integration
- `docs/migration/openclaw.md` — Migration from OpenClaw
- `docs/plans/*.md` — Architecture design documents

### Skills Documentation
Each skill has its own `SKILL.md` following a standard format with frontmatter.

## PR Template

Located at `.github/PULL_REQUEST_TEMPLATE.md`:

**Sections**:
- What does this PR do?
- Related Issue
- Type of Change (bug fix, feature, security fix, docs, tests, refactor, skill)
- Changes Made
- How to Test (with numbered steps)
- Checklist (Code + Documentation & Housekeeping)
- For New Skills (additional requirements)

## Issue Templates

Located at `.github/ISSUE_TEMPLATE/`:
- `bug_report.yml`
- `feature_request.yml`
- `setup_help.yml`
- `config.yml`

## Docs Deployment

- **CI**: `deploy-site.yml` workflow
- **Trigger**: Push to `main` with website changes
- **Process**: Build Docusaurus, stage with landing page, deploy to GitHub Pages
- **Domain**: `hermes-agent.nousresearch.com`
- **CNAME**: Maintained in deployment step

## Documentation Quality

- **Diagram Linting**: `ascii-guard` checks diagrams in docs
- **Search**: Local search via `@easyops-cn/docusaurus-search-local`
- **Diagrams**: Mermaid support via `@docusaurus/theme-mermaid`
- **Versioning**: React 19.0.0, TypeScript 5.6.2

## Skills System Documentation

Skills are documented with:
- `SKILL.md` format (required for bundled skills)
- Frontmatter with name, description, version, author, license, platforms
- `required_environment_variables` for secure setup
- `metadata.hermes` for conditional activation (fallback_for, requires_toolsets)
- Trigger conditions, quick reference, procedure, pitfalls, verification
