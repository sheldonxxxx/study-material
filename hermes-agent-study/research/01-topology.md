# Project Topology: hermes-agent

**Repository:** `/Users/sheldon/Documents/claw/reference/hermes-agent`
**Project:** Self-improving AI agent built by Nous Research
**Language:** Python (primary), TypeScript (website/docs)

---

## Directory Tree Overview

```
hermes-agent/
├── agent/                    # Core agent logic
│   ├── anthropic_adapter.py # Anthropic API integration
│   ├── auxiliary_client.py  # Auxiliary client support
│   ├── context_compressor.py
│   ├── display.py            # Terminal display rendering
│   ├── insights.py           # Usage insights/analytics
│   ├── model_metadata.py
│   ├── prompt_builder.py
│   └── usage_pricing.py
│
├── hermes_cli/               # CLI implementation (primary interface)
│   ├── main.py               # CLI entry point (~165KB)
│   ├── gateway.py            # Gateway management
│   ├── config.py             # Configuration management
│   ├── commands.py           # CLI commands
│   ├── models.py             # Model provider handling
│   ├── skills_hub.py         # Skills hub integration
│   ├── tools_config.py
│   ├── setup.py              # Setup wizard
│   ├── browser_tool.py       # Browser/Playwright integration
│   ├── skills_tool.py        # Skills management tool
│   ├── terminal_tool.py      # Terminal execution
│   ├── file_operations.py    # File operations
│   ├── rl_training_tool.py   # RL training tool
│   ├── mcp_tool.py           # MCP server integration
│   ├── voice_mode.py         # Voice/tts tools
│   ├── web_tools.py
│   ├── vision_tools.py
│   ├── memory_tool.py
│   ├── delegate_tool.py      # Subagent delegation
│   ├── skills_guard.py
│   └── environments/         # Runtime environments
│
├── gateway/                  # Messaging gateway
│   ├── run.py                # Gateway runner (~262KB)
│   ├── session.py            # Session management
│   ├── config.py             # Gateway config
│   ├── channel_directory.py
│   ├── delivery.py
│   ├── platforms/            # Platform integrations
│   │   ├── discord.py        # Discord bot
│   │   ├── telegram.py       # Telegram bot
│   │   ├── slack.py          # Slack integration
│   │   ├── whatsapp.py
│   │   ├── email.py
│   │   ├── matrix.py
│   │   ├── mattermost.py
│   │   ├── dingtalk.py
│   │   ├── signal.py
│   │   ├── sms.py
│   │   ├── homeassistant.py
│   │   └── webhook.py
│   └── platforms/            # Platform base classes
│
├── skills/                   # Skill definitions (35 categories)
│   ├── apple/
│   ├── autonomous-ai-agents/
│   ├── creative/
│   ├── data-science/
│   ├── diagramming/
│   ├── dogfood/
│   ├── domain/
│   ├── email/
│   ├── feeds/
│   ├── gaming/
│   ├── gifs/
│   ├── github/               # GitHub integration
│   ├── index-cache/
│   ├── inference-sh/
│   ├── leisure/
│   ├── mcp/
│   ├── media/
│   ├── mlops/                # MLOps tools
│   ├── music-creation/
│   ├── note-taking/
│   ├── productivity/
│   ├── red-teaming/
│   ├── research/
│   ├── smart-home/
│   ├── social-media/
│   └── software-development/
│
├── tools/                    # Tool implementations (~50 tools)
│   ├── browser_providers/
│   ├── environments/
│   ├── neutts_samples/
│   ├── file_operations.py
│   ├── web_tools.py
│   ├── mcp_tool.py
│   ├── code_execution_tool.py
│   └── ...
│
├── environments/             # Agent training/testing environments
│   ├── hermes_base_env.py
│   ├── agentic_opd_env.py
│   ├── web_research_env.py
│   ├── hermes_swe_env/
│   ├── terminal_test_env/
│   ├── tool_call_parsers/
│   └── benchmarks/
│
├── tests/                    # Test suite
│   ├── agent/
│   ├── gateway/
│   ├── hermes_cli/
│   ├── skills/
│   ├── integration/
│   ├── test_*.py             # Individual test files
│
├── acp_adapter/              # Adapter protocol
│   ├── __main__.py
│   ├── auth.py
│   ├── entry.py
│   ├── events.py
│   ├── permissions.py
│   ├── server.py
│   ├── session.py
│   └── tools.py
│
├── acp_registry/             # Skill registry
│   ├── autonomous-ai-agents/
│   ├── blockchain/
│   ├── creative/
│   ├── devops/
│   ├── email/
│   ├── health/
│   ├── mcp/
│   ├── migration/
│   ├── productivity/
│   ├── research/
│   └── security/
│
├── cron/                     # Cron scheduling
│   ├── jobs.py
│   └── scheduler.py
│
├── optional-skills/          # Optional skill packages
│   ├── agent.json
│   └── icon.svg
│
├── scripts/                  # Utility scripts
│   ├── install.sh            # Installation script
│   ├── install.ps1
│   ├── release.py
│   ├── hermes-gateway
│   └── whatsapp-bridge/
│
├── docs/                     # Documentation specs
│   ├── honcho-integration-spec.md
│   └── migration/
│
├── website/                  # Public website (static)
│   └── static assets
│
├── landingpage/              # Documentation site (Docusaurus)
│   ├── docs/
│   ├── src/
│   ├── docusaurus.config.ts
│   └── package.json
│
├── .plans/                   # Planning documents
├── honcho_integration/       # Honcho integration
├── nix/                      # Nix flake
├── datagen-config-examples/  # Data generation configs
├── assets/                   # Static assets
│
├── cli.py                    # Main CLI entry (~329KB)
├── run_agent.py              # Agent runner (~387KB)
├── rl_cli.py                 # RL CLI
├── hermes                    # Entry point wrapper script
├── hermes_state.py           # State management
├── hermes_constants.py
├── hermes_time.py
├── hermes_cli.py             # CLI module
├── model_tools.py
├── batch_runner.py           # Batch processing
├── mini_swe_runner.py
├── trajectory_compressor.py
├── toolset_distributions.py
├── toolsets.py
├── setup-hermes.sh
├── pyproject.toml
├── package.json
├── flake.nix
├── .env.example
├── cli-config.yaml.example
├── AGENTS.md
├── CONTRIBUTING.md
├── README.md
└── RELEASE_v*.md              # Release notes
```

---

## Entry Point Identification

| Entry Point | Purpose | Type |
|-------------|---------|------|
| `./hermes` | Convenience launcher script | Shell script |
| `cli.py` (~329KB) | Main CLI entry point, `fire.Fire(main)` | Python |
| `run_agent.py` (~387KB) | Agent execution logic | Python |
| `rl_cli.py` (~16KB) | Reinforcement learning CLI | Python |
| `hermes_cli/main.py` (~165KB) | Full CLI implementation | Python |
| `gateway/run.py` (~262KB) | Messaging gateway runner | Python |
| `scripts/hermes-gateway` | Gateway script | Shell |

---

## Key Files and Their Purposes

### Core Agent Files
| File | Purpose |
|------|---------|
| `agent/anthropic_adapter.py` | Anthropic API client integration |
| `agent/auxiliary_client.py` | Auxiliary client support for extended capabilities |
| `agent/context_compressor.py` | Context compression for long conversations |
| `agent/prompt_builder.py` | Prompt construction and management |
| `agent/display.py` | Terminal display rendering |
| `agent/insights.py` | Usage analytics and insights |
| `agent/usage_pricing.py` | Cost tracking and pricing |

### CLI Files
| File | Purpose |
|------|---------|
| `hermes_cli/main.py` | Main CLI application (~165KB) |
| `hermes_cli/gateway.py` | Gateway management |
| `hermes_cli/config.py` | Configuration handling (~76KB) |
| `hermes_cli/commands.py` | CLI command definitions |
| `hermes_cli/models.py` | Model provider selection |
| `hermes_cli/setup.py` | Setup wizard (~139KB) |
| `hermes_cli/browser_tool.py` | Browser automation via Playwright |
| `hermes_cli/skills_hub.py` | Skills hub integration |

### Gateway Files
| File | Purpose |
|------|---------|
| `gateway/run.py` | Gateway main runner (~262KB) |
| `gateway/session.py` | Session management |
| `gateway/platforms/discord.py` | Discord bot (~94KB) |
| `gateway/platforms/telegram.py` | Telegram bot (~78KB) |
| `gateway/platforms/slack.py` | Slack integration |
| `gateway/platforms/whatsapp.py` | WhatsApp integration |
| `gateway/platforms/email.py` | Email integration |

### Runner/Execution Files
| File | Purpose |
|------|---------|
| `run_agent.py` | Core agent execution (~387KB) |
| `batch_runner.py` | Batch trajectory processing |
| `mini_swe_runner.py` | SWE-benchmark runner |
| `trajectory_compressor.py` | Trajectory compression for training |

---

## Directory Purpose Mapping

| Directory | Purpose |
|----------|---------|
| `agent/` | Core agent logic - adapters, context management, prompts, display |
| `hermes_cli/` | CLI implementation - commands, configuration, tools, skins |
| `gateway/` | Messaging gateway - multi-platform bot integrations |
| `skills/` | Skill definitions organized by domain/category |
| `tools/` | Tool implementations - file ops, web, code execution, etc. |
| `environments/` | Agent environments for training/evaluation |
| `tests/` | Comprehensive test suite |
| `acp_adapter/` | Adapter protocol implementation |
| `acp_registry/` | Skill registry by category |
| `cron/` | Cron job scheduling |
| `optional-skills/` | Optional skill packages |
| `scripts/` | Installation and utility scripts |
| `docs/` | Documentation specifications |
| `website/` | Public marketing website |
| `landingpage/` | Documentation site (Docusaurus) |
| `.plans/` | Project planning documents |

---

## Architecture Patterns

- **CLI-first design**: Primary interface via `hermes_cli/`, with gateway as secondary
- **Multi-platform gateway**: Unified messaging across Discord, Telegram, Slack, WhatsApp, Signal, Email, Matrix, etc.
- **Tool-based execution**: Extensive tool system in `hermes_cli/` and `tools/`
- **Skill system**: Procedural memory and skill creation in `skills/`
- **Environment abstraction**: Multiple execution backends (local, Docker, SSH, Daytona, Modal)
- **RL-ready**: Batch runners and trajectory compression for training
