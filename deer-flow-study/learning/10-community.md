# Community

## Overview

DeerFlow is an open-source project developed and maintained by ByteDance's Volcengine team. The project launched version 2.0 in February 2026 and quickly gained significant traction, reaching #1 on GitHub Trending on February 28, 2026.

## Project Status

- **Version**: 2.0 (ground-up rewrite, no code shared with 1.x)
- **License**: MIT
- **Repository**: [github.com/bytedance/deer-flow](https://github.com/bytedance/deer-flow)
- **Website**: [deerflow.tech](https://deerflow.tech)
- **Current Branch**: `main` (2.0 development)
- **Legacy Branch**: `main-1.x` (original Deep Research framework, still maintained)

## Trending Achievement

On February 28, 2026, DeerFlow reached **#1 on GitHub Trending** following the launch of version 2.0. The project gained significant community attention for its super agent harness architecture.

## Core Team

### Key Contributors

| Name | GitHub | Role |
|------|--------|------|
| Daniel Walnut | [@hetaoBackend](https://github.com/hetaoBackend/) | Core author |
| Henry Li | [@magiccube](https://github.com/magiccube/) | Core author |

Both core authors are affiliated with ByteDance Volcengine.

## Affiliation

DeerFlow is a **ByteDance Volcengine** project. The official website and documentation are hosted through Volcengine services.

### Coding Plan Partnership

DeerFlow strongly recommends using:
- **Doubao-Seed-2.0-Code**
- **DeepSeek v3.2**
- **Kimi 2.5**

For running DeerFlow, in partnership with BytePlus coding plan initiatives.

### InfoQuest Integration

DeerFlow integrates **InfoQuest**, BytePlus's intelligent search and crawling toolset, providing free online experience for web research capabilities.

## Contributing

### How to Contribute

1. **Fork and Branch**: Create feature branches from `main`
2. **Development**: Follow the Docker (recommended) or local development setup
3. **Testing**: Ensure all tests pass (`make test` for backend, `pnpm check` for frontend)
4. **Documentation**: Update README.md and CLAUDE.md after code changes
5. **Commit Style**: Use conventional commits (e.g., `feat:`, `fix:`, `docs:`)
6. **Pull Request**: Push and create PR against `main`

### Contribution Guidelines

- **Docker Recommended**: For consistent development environment
- **TDD Required**: Every feature or bug fix must include unit tests
- **Code Style**:
  - Backend: ruff (Python)
  - Frontend: ESLint + Prettier (TypeScript)
- **Regression Coverage**: PRs trigger backend unit tests via GitHub Actions
- **CLAUDE.md Policy**: Documentation must be synchronized after code changes

### Quick Start for Contributors

```bash
# Clone and configure
git clone https://github.com/bytedance/deer-flow.git
cd deer-flow
make config

# Docker development (recommended)
make docker-init
make docker-start

# Or local development
make check    # Verify prerequisites
make install   # Install dependencies
make dev       # Start all services
```

## Community Resources

| Resource | Link |
|----------|------|
| GitHub Issues | [Issues](https://github.com/bytedance/deer-flow/issues) |
| GitHub Discussions | [Discussions](https://github.com/bytedance/deer-flow/discussions) |
| Security Reports | [Security Policy](SECURITY.md) |

## International Support

The project provides multi-language README translations:
- English (primary)
- Chinese (README_zh.md)
- Japanese (README_ja.md)
- French (README_fr.md)
- Russian (README_ru.md)

## Acknowledgments

DeerFlow explicitly acknowledges and感谢 these open-source projects:

### Primary Dependencies

| Project | Purpose |
|---------|---------|
| [LangChain](https://github.com/langchain-ai/langchain) | LLM interactions and chains framework |
| [LangGraph](https://github.com/langchain-ai/langgraph) | Multi-agent orchestration |

### Key Technologies

- **LangGraph** -powers the agent workflow orchestration
- **LangChain** - enables seamless LLM integration
- **FastAPI** - REST API framework
- **Next.js** - React framework
- **Radix UI** - Accessible component primitives
- **Tailwind CSS** - Utility-first styling

## Architecture Philosophy

DeerFlow's architecture emphasizes:

1. **Extensibility**: MCP servers, skills system, and tool plugins
2. **Isolation**: Sub-agents run in isolated contexts; sandboxes provide execution isolation
3. **Memory**: Long-term memory across sessions
4. **Model Agnostic**: Works with any OpenAI-compatible API
5. **Production-Ready**: Comprehensive testing and documentation

## Version History

### v2.0 (Current)
- Ground-up rewrite
- Super agent harness architecture
- LangGraph-based orchestration
- MCP and ACP tool protocols
- Docker/Kubernetes sandbox support
- IM channel integrations (Telegram, Slack, Feishu)

### v1.x (Legacy)
- Original Deep Research framework
- Still maintained on `main-1.x` branch
- Contributions welcome but limited new features

## Star History

The project tracks its GitHub star growth via Star History. The trajectory shows significant growth following the v2.0 launch and GitHub Trending achievement.
