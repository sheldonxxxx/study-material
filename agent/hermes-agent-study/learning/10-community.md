# Community - Hermes Agent

## Project Overview

**Hermes Agent** is built by [Nous Research](https://nousresearch.com), the AI lab behind the Hermes, Nomos, and Psyche models.

**Repository**: [github.com/NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)

**License**: MIT

## Built By

**Nous Research** — AI research lab focused on:
- Hermes models (LLM series)
- Nomos models
- Psyche models
- Agent frameworks and tooling

Website: [nousresearch.com](https://nousresearch.com)
Nous Portal: [portal.nousresearch.com](https://portal.nousresearch.com)

## Community Channels

| Channel | Purpose | Link |
|---------|---------|------|
| Discord | Questions, showcasing projects, skill sharing | [discord.gg/NousResearch](https://discord.gg/NousResearch) |
| GitHub Issues | Bug reports, feature requests | [github.com/NousResearch/hermes-agent/issues](https://github.com/NousResearch/hermes-agent/issues) |
| GitHub Discussions | Design proposals, architecture discussions | [github.com/NousResearch/hermes-agent/discussions](https://github.com/NousResearch/hermes-agent/discussions) |
| Skills Hub | Community skill sharing | [agentskills.io](https://agentskills.io) |

## Skills Hub

Hermes supports the open [agentskills.io](https://agentskills.io) standard for skill sharing.

**Skill Distribution**:
- **Bundled skills** (`skills/`) — Ship with every install, broadly useful
- **Optional skills** (`optional-skills/`) — Official but not activated by default
- **Hub skills** — Community-contributed, installable via `hermes skills install`

**Skill Discovery**: `hermes skills browse` shows all available skills with labels (official, community)

## Contributing

### Contribution Priorities (in order)

1. Bug fixes — crashes, incorrect behavior, data loss
2. Cross-platform compatibility — Windows, macOS, Linux
3. Security hardening — shell injection, prompt injection, path traversal
4. Performance and robustness — retry logic, error handling
5. New skills — broadly useful ones only
6. New tools — rarely needed, most should be skills
7. Documentation — fixes, clarifications, examples

### Development Setup

```bash
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
uv venv venv --python 3.11
source venv/bin/activate
uv pip install -e ".[all,dev]"
pytest tests/ -v
```

### Branch Naming

```
fix/description        # Bug fixes
feat/description       # New features
docs/description       # Documentation
test/description       # Tests
refactor/description   # Code restructuring
```

### Commit Message Format

**Conventional Commits**: `<type>(<scope>): <description>`

Types: `fix`, `feat`, `docs`, `test`, `refactor`, `chore`
Scopes: `cli`, `gateway`, `tools`, `skills`, `agent`, `install`, `whatsapp`, `security`, etc.

### Pull Request Process

1. Run tests: `pytest tests/ -v`
2. Test manually on your platform
3. Keep PRs focused (one logical change per PR)
4. Fill out PR template completely
5. Tag maintainers for review

### PR Checklist

**Code**:
- [ ] Read Contributing Guide
- [ ] Commit messages follow Conventional Commits
- [ ] No duplicate PRs
- [ ] PR contains only related changes
- [ ] `pytest tests/ -q` passes
- [ ] Added tests for changes
- [ ] Tested on your platform

**Documentation & Housekeeping**:
- [ ] Updated relevant documentation
- [ ] Updated `cli-config.yaml.example` if needed
- [ ] Updated `CONTRIBUTING.md` or `AGENTS.md` if architecture changed
- [ ] Considered cross-platform impact
- [ ] Updated tool descriptions/schemas if needed

### For New Skills

- [ ] Broadly useful to most users (if bundled)
- [ ] SKILL.md follows standard format
- [ ] No external dependencies not already available
- [ ] Tested end-to-end

## Issue Templates

Located at `.github/ISSUE_TEMPLATE/`:
- `bug_report.yml` — Bug reports with OS, Python version, Hermes version, traceback
- `feature_request.yml` — Feature requests
- `setup_help.yml` — Installation/setup help
- `config.yml` — Issue configuration

## Skills Creation Guidelines

### When to Make it a Skill

- Capability expressed as instructions + shell commands + existing tools
- Wraps external CLI/API callable via `terminal` or `web_extract`
- No custom Python integration or API key management needed

### When to Make it a Tool

- Requires end-to-end API key/auth flow integration
- Needs custom processing that must execute precisely every time
- Handles binary data, streaming, or real-time events

### Skill Format

SKILL.md with YAML frontmatter:
```yaml
---
name: skill-name
description: Brief description
version: 1.0.0
author: Your Name
license: MIT
platforms: [macos, linux]  # Optional
required_environment_variables:
  - name: API_KEY
    prompt: API key
    help: Where to get it
---
```

### Skill Metadata

```yaml
metadata:
  hermes:
    tags: [Category, Subcategory, Keywords]
    related_skills: [other-skill]
    fallback_for_toolsets: [web]      # Show when toolset unavailable
    requires_toolsets: [terminal]     # Show only with toolset available
```

## Security

### Reporting Security Issues

- Report vulnerabilities privately to maintainers
- Security fixes get dedicated PR type: `🔒 Security fix`

### Security Considerations in Code

| Layer | Protection |
|-------|------------|
| Sudo piping | `shlex.quote()` prevents shell injection |
| Dangerous commands | Regex patterns in `tools/approval.py` |
| Cron injection | Scanner in `tools/cronjob_tools.py` |
| Path traversal | `os.path.realpath()` before access control |
| Skills guard | Security scanner for hub skills |
| Code sandbox | API keys stripped in `execute_code` |
| Container hardening | Docker: all capabilities dropped |

## Ecosystem Projects

### Integrations

| Project | Purpose |
|---------|---------|
| [Honcho](https://github.com/plastic-labs/honcho) | Dialectic user modeling |
| [ACP](https://github.com/anysphere/acp) | Agent Client Protocol for IDE integration |
| [agentskills.io](https://agentskills.io) | Open skill standard and registry |
| [firecrawl](https://firecrawl.dev) | Web scraping (via firecrawl-py) |
| [edge-tts](https://github.com/rany2/edge-tts) | Free TTS (no API key) |
| [faster-whisper](https://github.com/guillaumekln/faster-whisper) | Fast Whisper transcription |

### Compatible Services

| Service | Purpose |
|---------|---------|
| Nous Portal | Provider with OAuth |
| OpenRouter | 200+ models via single API |
| OpenAI | Direct API |
| Anthropic | Claude API |
| Kimi/Moonshot | Chinese LLM provider |
| MiniMax | Chinese LLM provider |
| z.ai/GLM | Chinese LLM provider |

### Terminal Backends

| Backend | Provider |
|---------|----------|
| Local | Native |
| Docker | Containerization |
| SSH | Remote execution |
| Singularity | HPC containers |
| Modal | Serverless (GPU) |
| Daytona | Cloud dev environments |

## Events & Releases

### Release Notes

- `RELEASE_v0.2.0.md` — v0.2.0 release notes
- `RELEASE_v0.3.0.md` — v0.3.0 release notes
- Version in `pyproject.toml` (`project.version`)

### Version Support

- Current: Active development on main branch
- Python: 3.11+
- Platforms: Linux, macOS, Windows (WSL2)

## Code Quality

- **Style**: PEP 8 (practical exceptions for line length)
- **Commits**: Conventional Commits enforced via PR template
- **Testing**: pytest with pytest-asyncio, pytest-xdist for parallel
- **Type Safety**: TypeScript for website, Python type hints encouraged
- **Cross-Platform**: Required consideration in PR checklist

## Documentation Standards

- Docusaurus 3.9.2 site at `hermes-agent.nousresearch.com/docs`
- In-repo docs for setup and migration
- SKILL.md format required for bundled skills
- PR template includes documentation checklist

## Support

| Resource | Use Case |
|----------|----------|
| Discord | Questions, showcases, skill sharing |
| GitHub Issues | Bugs, feature requests |
| GitHub Discussions | Architecture, design proposals |
| `/hermes doctor` | Self-diagnostic in CLI |
| `hermes --help` | CLI reference |
