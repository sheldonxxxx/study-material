# OpenFang Project Overview

## What Is OpenFang?

OpenFang is an **open-source Agent Operating System** built entirely in Rust -- not a chatbot framework, not a Python wrapper around an LLM, and not a "multi-agent orchestrator." It is a full operating system for autonomous agents that runs continuously, executing tasks on schedules, building knowledge graphs, monitoring targets, generating leads, managing social media, and reporting results to a dashboard.

The entire system compiles to a **single ~32MB binary**. One install, one command, agents are live.

## Key Metrics

| Metric | Value |
|--------|-------|
| **Lines of Code** | 137,728 LOC |
| **Rust Crates** | 14 crates |
| **Test Count** | 1,767+ tests passing |
| **Clippy Warnings** | 0 (zero warnings enforced) |
| **Install Size** | ~32 MB |
| **Cold Start Time** | <200ms |
| **Idle Memory** | ~40 MB |
| **Built-in Tools** | 53 + MCP + A2A |
| **Channel Adapters** | 40 messaging platforms |
| **LLM Providers** | 27 providers, 123+ models |
| **Bundled Skills** | 60 skills |
| **Autonomous Hands** | 7 pre-built agents |
| **Security Systems** | 16 discrete layers |
| **License** | MIT |
| **Current Version** | 0.3.30 (March 2026) |

## Core Innovation: Hands

Traditional agent frameworks wait for user input. **Hands** are OpenFang's core innovation -- pre-built autonomous capability packages that run independently, on schedules, without prompting. Each Hand bundles:

- **HAND.toml** -- Manifest declaring tools, settings, requirements, and dashboard metrics
- **System Prompt** -- Multi-phase operational playbook (500+ word expert procedures)
- **SKILL.md** -- Domain expertise reference injected into context at runtime
- **Guardrails** -- Approval gates for sensitive actions

### The 7 Bundled Hands

| Hand | Capability |
|------|------------|
| **Clip** | YouTube video download, moment identification, vertical shorts cutting, caption generation, AI voice-over, publishing to Telegram/WhatsApp. 8-phase FFmpeg + yt-dlp pipeline. |
| **Lead** | Daily prospect discovery, web enrichment, 0-100 scoring, deduplication, CSV/JSON/Markdown delivery. Builds ICP profiles over time. |
| **Collector** | OSINT-grade intelligence. Continuous monitoring, change detection, sentiment tracking, knowledge graph construction, critical shift alerts. |
| **Predictor** | Superforecasting engine. Multi-source signal collection, calibrated reasoning chains, confidence intervals, Brier score accuracy tracking, contrarian mode. |
| **Researcher** | Deep autonomous research. Cross-referencing, CRAAP credibility evaluation, APA-formatted cited reports, multi-language support. |
| **Twitter** | Autonomous X/Twitter management. 7 rotating content formats, optimal scheduling, mention responses, performance tracking, approval queue. |
| **Browser** | Web automation agent. Multi-step workflow navigation, form filling, button clicking, Playwright bridge with session persistence. Mandatory purchase approval gate. |

## Architecture: 14 Crates

```
openfang-kernel      Orchestration, workflows, metering, RBAC, scheduler, budget tracking
openfang-runtime     Agent loop, 3 LLM drivers, 53 tools, WASM sandbox, MCP, A2A
openfang-api         140+ REST/WS/SSE endpoints, OpenAI-compatible API, dashboard
openfang-channels    40 messaging adapters with rate limiting, DM/group policies
openfang-memory      SQLite persistence, vector embeddings, canonical sessions, compaction
openfang-types       Core types, taint tracking, Ed25519 manifest signing, model catalog
openfang-skills      60 bundled skills, SKILL.md parser, FangHub marketplace
openfang-hands       7 autonomous Hands, HAND.toml parser, lifecycle management
openfang-extensions  25 MCP templates, AES-256-GCM credential vault, OAuth2 PKCE
openfang-wire        OFP P2P protocol with HMAC-SHA256 mutual authentication
openfang-cli         CLI with daemon management, TUI dashboard, MCP server mode
openfang-desktop     Tauri 2.0 native app (system tray, notifications, global shortcuts)
openfang-migrate     OpenClaw, LangChain, AutoGPT migration engine
xtask                Build automation
```

## Competitive Positioning

OpenFang benchmarks against the agent framework landscape:

| Framework | Language | Cold Start | Install Size | Security Score | Channel Adapters |
|-----------|----------|------------|--------------|----------------|------------------|
| **OpenFang** | Rust | <200ms | ~32 MB | 16 systems | 40 |
| ZeroClaw | Rust | ~10ms | ~8.8 MB | 6 layers | 15 |
| OpenClaw | TypeScript | ~6s | ~500 MB | 3 basic | 13 |
| LangGraph | Python | ~2.5s | ~150 MB | 2 basic | 0 |
| CrewAI | Python | ~3s | ~100 MB | 1 basic | 0 |
| AutoGen | Python | ~4s | ~200 MB | 2 basic | 0 |

## 40 Channel Adapters

**Core:** Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Email (IMAP/SMTP)

**Enterprise:** Microsoft Teams, Mattermost, Google Chat, Webex, Feishu/Lark, Zulip

**Social:** LINE, Viber, Facebook Messenger, Mastodon, Bluesky, Reddit, LinkedIn, Twitch

**Community:** IRC, XMPP, Guilded, Revolt, Keybase, Discourse, Gitter

**Privacy:** Threema, Nostr, Mumble, Nextcloud Talk, Rocket.Chat, Ntfy, Gotify

**Workplace:** Pumble, Flock, Twist, DingTalk, Zalo, Webhooks

## 27 LLM Providers

Anthropic, Gemini, OpenAI, Groq, DeepSeek, OpenRouter, Together, Mistral, Fireworks, Cohere, Perplexity, xAI, AI21, Cerebras, SambaNova, HuggingFace, Replicate, Ollama, vLLM, LM Studio, Qwen, MiniMax, Zhipu, Moonshot, Qianfan, Bedrock, and more.

## Target Users

1. **Developers** wanting a production-ready agent framework with battle-tested security
2. **Builders** who need autonomous agents that work 24/7 on schedules
3. **Teams** requiring multi-channel presence (40 adapters) with unified agent management
4. **Enterprises** needing RBAC, audit trails, and credential vault capabilities
5. **Individuals** who want personal agents for lead generation, research, social media management

## Stability Status

OpenFang v0.3.30 is pre-1.0. The architecture is solid, the test suite is comprehensive, and the security model is thorough. Until v1.0:

- Breaking changes may occur between minor versions
- Some Hands are more mature than others (Browser and Researcher are most battle-tested)
- Edge cases exist -- report issues at github.com/RightNow-AI/openfang/issues
- Pin to a specific commit for production deployments

## Development Practices

```bash
cargo build --workspace --lib          # Must compile
cargo test --workspace                 # All 1,767+ tests must pass
cargo clippy --workspace --all-targets -- -D warnings  # Zero warnings
cargo fmt --all -- --check             # Format check
```

## Links

- Website: https://openfang.sh
- Documentation: https://openfang.sh/docs
- GitHub: https://github.com/RightNow-AI/openfang
- Discord: https://discord.gg/sSJqgNnq6X
- Twitter: https://x.com/openfangg
- Security Contact: jaber@rightnowai.co
