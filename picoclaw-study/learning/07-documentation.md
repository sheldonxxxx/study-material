# Documentation

## Official Documentation

- **Website**: [picoclaw.io](https://picoclaw.io) — Auto-detects platform and provides one-click downloads
- **Docs**: [docs.picoclaw.io](https://docs.picoclaw.io) — Official documentation site
- **DeepWiki**: [deepwiki.com/sipeed/picoclaw](https://deepwiki.com/sipeed/picoclaw) — AI-powered codebase wiki
- **GitHub Releases**: [github.com/sipeed/picoclaw/releases](https://github.com/sipeed/picoclaw/releases) — Binary downloads

## README Files

The project provides multilingual README translations in the repository root:

| File | Language |
|------|----------|
| `README.md` | English (primary) |
| `README.zh.md` | Simplified Chinese |
| `README.ja.md` | Japanese |
| `README.pt-br.md` | Brazilian Portuguese |
| `README.vi.md` | Vietnamese |
| `README.fr.md` | French |
| `README.it.md` | Italian |
| `README.id.md` | Indonesian |

## Documentation Directory (`docs/`)

```
docs/
  chat-apps.md              # All 17+ channel setup guides
  configuration.md          # Environment variables, workspace layout, security sandbox
  credential_encryption.md # Sensitive data handling
  debug.md                  # Debugging guide
  docker.md                 # Docker & Quick Start
  hardware-compatibility.md # Tested boards and minimum requirements
  hooks/                    # Hook system documentation
    README.md
  providers.md              # 30+ LLM providers, model routing
  security_configuration.md # Security best practices
  sensitive_data_filtering.md # Data filtering
  spawn-tasks.md            # Quick tasks, long tasks, async sub-agent orchestration
  steering.md               # Inject messages into running agent loop
  subturn.md                # Subagent coordination, concurrency control
  tools_configuration.md    # Per-tool config, exec policies, MCP, Skills
  troubleshooting.md         # Common issues and solutions
  config-versioning.md      # Config migration between versions
  channels/                 # Channel-specific documentation
    telegram/README.md
    discord/README.md
    qq/README.md
    slack/README.md
    matrix/README.md
    dingtalk/README.md
    feishu/README.md
    onebot/README.md
    maixcam/README.md
  design/                   # Architectural design docs
  migration/                # Version migration guides
  zh/ ja/ fr/ it/ pt-br/ vi/  # Localized docs
```

## CONTRIBUTING.md

The project has a comprehensive `CONTRIBUTING.md` at the repository root covering:

- Code of Conduct
- Development setup (Go 1.25+, `make`)
- Build and test commands (`make build`, `make check`, `make test`)
- Branch strategy (`main` for development, `release/x.y` for stable)
- AI-assisted contribution guidelines (disclosure required, same quality bar)
- PR process (template, size limits, squash merge)
- Code review guidelines and reviewer assignments by function

The CONTRIBUTING guide explicitly states PicoClaw was AI-bootstrapped and welcomes AI-assisted contributions with honest disclosure.

## ROADMAP.md

Public roadmap available at [github.com/sipeed/picoclaw/issues/988](https://github.com/sipeed/picoclaw/issues/988).

## API and SDK Documentation

- **MCP (Model Context Protocol)**: Native support via `github.com/modelcontextprotocol/go-sdk`; configurable in `tools.mcp` section of config
- **Providers**: 30+ LLM providers documented in `docs/providers.md` with `protocol/model` format
- **Channels**: 17+ chat platforms documented in `docs/chat-apps.md` and channel-specific READMEs

## Skill System Documentation

Skills are documented in `docs/tools_configuration.md#skills-tool`:
- Installed from ClawHub registry
- Configured via `tools.skills` in config.json
- Loaded from `SKILL.md` files in workspace

## Architecture Documentation

- `docs/design/` — Architectural design documents
- `pkg/` directory structure is self-documenting with packages for each subsystem
- Code is intentionally small and readable (per README)

## Security Documentation

- `docs/security_configuration.md` — Security best practices
- `docs/credential_encryption.md` — Sensitive credential handling
- `docs/sensitive_data_filtering.md` — Data filtering
- Config version migration: `docs/config-versioning.md`

## Community Resources

- Discord: [discord.gg/V4sAZ9XWpN](https://discord.gg/V4sAZ9XWpN)
- WeChat group (QR code in README and `assets/wechat.png`)
- GitHub Issues for bug reports and feature requests
- GitHub Discussions for general questions and ideas

## Quick Reference

| Resource | Link |
|----------|------|
| Official site | [picoclaw.io](https://picoclaw.io) |
| Docs | [docs.picoclaw.io](https://docs.picoclaw.io) |
| GitHub | [github.com/sipeed/picoclaw](https://github.com/sipeed/picoclaw) |
| Community roadmap | [#988](https://github.com/sipeed/picoclaw/issues/988) |
| Discord | [discord.gg/V4sAZ9XWpN](https://discord.gg/V4sAZ9XWpN) |
