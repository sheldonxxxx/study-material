# Tech Stack Analysis: Hermes Agent

## Language and Runtime Versions

| Component | Version | Notes |
|-----------|---------|-------|
| Python | >= 3.11 | Primary language, enforced in `pyproject.toml` |
| Node.js | >= 18 (root), >= 20 (website) | For browser tools and website |
| Nix | nixos-24.11 channel | Optional but officially supported |

## Core Python Dependencies

### AI Model Integration
| Package | Version | Purpose |
|---------|---------|---------|
| `openai` | >= 2.21.0, < 3 | OpenAI API client |
| `anthropic` | >= 0.39.0, < 1 | Anthropic API client (Claude) |
| `fal-client` | >= 0.13.1, < 1 | Fal.ai integration |

### CLI and User Interface
| Package | Version | Purpose |
|---------|---------|---------|
| `fire` | >= 0.7.1, < 1 | CLI argument parsing |
| `prompt_toolkit` | >= 3.0.52, < 4 | Interactive CLI |
| `rich` | >= 14.3.3, < 15 | Terminal formatting |
| `jinja2` | >= 3.1.5, < 4 | Template rendering |

### Data and Networking
| Package | Version | Purpose |
|---------|---------|---------|
| `httpx` | >= 0.28.1, < 1 | HTTP client |
| `requests` | >= 2.33.0, < 3 | HTTP library (pinned for CVE-2026-25645) |
| `pydantic` | >= 2.12.5, < 3 | Data validation |
| `pyyaml` | >= 6.0.2, < 7 | YAML config parsing |

### Reliability and Utilities
| Package | Version | Purpose |
|---------|---------|---------|
| `tenacity` | >= 9.1.4, < 10 | Retry logic |
| `python-dotenv` | >= 1.2.1, < 2 | .env file loading |
| `PyJWT[crypto]` | >= 2.12.0, < 3 | JWT auth (pinned for CVE-2026-32597) |

### Tools and Browser Automation
| Package | Version | Purpose |
|---------|---------|---------|
| `agent-browser` | ^0.13.0 | Browser automation (Node.js package) |
| `firecrawl-py` | >= 4.16.0, < 5 | Web scraping |
| `parallel-web` | >= 0.4.2, < 1 | Parallel web requests |

### Voice and Audio
| Package | Version | Purpose |
|---------|---------|---------|
| `edge-tts` | >= 7.2.7, < 8 | Free TTS (Edge) |
| `faster-whisper` | >= 1.0.0, < 2 | Whisper transcription |

### Optional Extras (via extras)
| Extra | Key Dependencies |
|-------|------------------|
| `dev` | pytest, pytest-asyncio, pytest-xdist, mcp |
| `messaging` | python-telegram-bot, discord.py, slack-bolt, aiohttp |
| `mcp` | MCP (Model Context Protocol) client |
| `honcho` | honcho-ai >= 2.0.1, < 3 |
| `voice` | sounddevice, numpy |
| `cli` | simple-term-menu |
| `acp` | agent-client-protocol >= 0.8.1, < 1.0 |
| `rl` | FastAPI, uvicorn, wandb, atroposlib, tinker |
| `modal` | swe-rex[modal] |
| `daytona` | daytona >= 0.148.0, < 1 |

## Build System and Tooling

### Primary Package Manager: uv
- **uv** is the primary package manager (installed via `curl -LsSf https://astral.sh/uv/install.sh | sh`)
- Uses `pyproject.toml` for dependency specification (PEP 621 standard)
- `uv.lock` file for reproducible builds

### Build Backend
- **setuptools** >= 61.0 with `setuptools.build_meta` as build backend
- Editable installs via `uv pip install -e ".[all]"`

### Nix Integration
The project provides optional Nix-based development environments:

| File | Purpose |
|------|---------|
| `flake.nix` | Nix flake configuration |
| `nix/python.nix` | uv2nix virtual environment builder |
| `nix/devShell.nix` | Fast dev shell with stamp-file optimization |
| `nix/packages.nix` | Package derivation for distribution |
| `nix/checks.nix` | Nix checks/tests |
| `nix/nixosModules.nix` | NixOS module definitions |

### Entry Points
```
hermes = hermes_cli.main:main
hermes-agent = run_agent:main
hermes-acp = acp_adapter.entry:main
```

## CI/CD Configuration

### GitHub Actions Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `tests.yml` | Push/PR to main | Run pytest with uv on Python 3.11 |
| `nix.yml` | Push to main, PR (flake.nix, pyproject.toml, etc.) | Nix flake check on Linux, evaluate on macOS |
| `deploy-site.yml` | Push to main (website/**, landingpage/**) | Build and deploy Docusaurus to GitHub Pages |
| `docs-site-checks.yml` | PR on website/** | Lint diagrams, build Docusaurus |
| `supply-chain-audit.yml` | PR opened/sync/reopened | Scan for supply chain attack patterns (.pth, base64+exec, etc.) |

### Supply Chain Security
The `supply-chain-audit.yml` workflow scans PRs for:
- `.pth` files (auto-execute on Python startup)
- base64 decode + exec/eval patterns (litellm attack pattern)
- subprocess with encoded commands
- marshal/pickle/compile usage
- Network POST/PUT calls

## Project Structure

```
hermes-agent/
├── agent/              # Core agent logic
│   ├── anthropic_adapter.py
│   ├── copilot_acp_client.py
│   ├── prompt_builder.py
│   ├── smart_model_routing.py
│   └── ...
├── hermes_cli/         # CLI implementation (~35 modules)
├── tools/              # Tool implementations (~50 modules)
├── skills/             # Bundled skills (skill manifests + scripts)
├── environments/       # Benchmark/evaluation environments
├── gateway/            # Messaging gateway (Telegram, Discord, Slack, etc.)
├── acp_adapter/        # Agent Client Protocol adapter
├── website/            # Docusaurus documentation site
├── landingpage/        # Marketing landing page
├── nix/                # Nix configurations
├── tests/              # Python tests (~100 test files)
└── scripts/            # Utility scripts
```

## Documentation Site

- **Framework**: Docusaurus 3.9.2
- **Node.js**: >= 20
- **Features**: Mermaid diagrams, local search (easyops-cn/docusaurus-search-local)
- **Deployment**: GitHub Pages (hermes-agent.nousresearch.com)

## Notable Framework Choices

1. **uv for Python package management** - Astral's ultra-fast Rust-based package manager
2. **Nix flakes** - Reproducible builds and dev environments
3. **Agent Client Protocol (ACP)** - Nous Research's protocol for agent-to-agent communication
4. **Skills system** - Extensible skill manifests that can be synced to `~/.hermes/skills/`
5. **Multiple messaging platform support** - Telegram, Discord, Slack, WhatsApp, Matrix, SMS via unified gateway
6. **Browser automation** - Playwright-based via `agent-browser` package
7. **RL training integration** - Optional `atroposlib` and `tinker` for reinforcement learning workflows

## System Dependencies (from install.sh)

| Dependency | Purpose | Platform |
|-----------|---------|----------|
| `ripgrep` | Fast file search | Linux/macOS |
| `ffmpeg` | TTS voice messages | Linux/macOS |
| `git` | Version control | All |
| `node` | Browser tools runtime | All |
