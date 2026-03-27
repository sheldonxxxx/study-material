# Nanobot Study: Learning Summary

## What This Is

This folder contains synthesized learning documents distilled from deep research into the nanobot project (v0.1.4.post5). Nanobot is an ultra-lightweight personal AI assistant framework written in Python that connects LLM providers to multiple chat platforms.

**Research basis:** 11 research documents analyzing topology, architecture, tech stack, features, code quality, and security patterns across 36,000+ stars and 200+ contributors.

---

## Key Takeaways

1. **Event-driven architecture with async message bus** is the core pattern. Channels produce messages to a queue, AgentLoop consumes and processes, results flow back through the bus to channels.

2. **Plugin-based extensibility** everywhere: channels, providers, tools, skills all use registry patterns enabling runtime discovery without core code changes.

3. **Multi-provider abstraction is sophisticated** - 20+ LLMs supported via ProviderSpec registry with OpenAI-compatible as the dominant backend pattern.

4. **Tool security is defense-in-depth**: Shell execution uses regex deny-patterns + path restrictions + SSRF blocking. Web fetch validates resolved IPs against blocklist.

5. **Memory is two-layer**: MEMORY.md for LLM-summarized facts, HISTORY.md for append-only searchable log. Token budget drives automatic consolidation.

6. **Skills are markdown prompts, not code plugins** - Skills are SKILL.md files loaded into context, not compiled extensions. Installation is via ClawHub CLI (npx).

7. **WhatsApp requires TypeScript bridge** because WhatsApp Web needs Baileys library. Bundled into Python wheel via hatchling force-include.

---

## Should Trust / Should Not Trust

### Should Trust

- **Architecture**: Event-driven with message bus - well-proven, works at scale
- **Provider abstraction**: OpenAI-compatible pattern handles 20+ providers correctly
- **Security module (network.py)**: SSRF protection is comprehensive
- **Concurrency model**: Per-session locks with cross-session parallelism is correct
- **Retry logic**: Exponential backoff for transient failures is properly implemented

### Should Not Trust

- **"Social network" feature**: Not a native protocol - just skill.md files read via web_fetch
- **Multi-instance global state**: `_current_config_path` is not thread-safe
- **Shell deny patterns**: Regex-based blocking is not a true sandbox
- **Memory has no delete**: Facts persist indefinitely in MEMORY.md
- **OAuth providers**: OpenAI Codex/GitHub Copilot have `is_oauth=True` but flow incomplete

---

## At a Glance

| Aspect | Details |
|--------|---------|
| **Language** | Python 3.11+ (core), TypeScript (WhatsApp bridge) |
| **Architecture** | Event-driven with async message bus |
| **Key Libraries** | Pydantic v2, Loguru, httpx, croniter, tiktoken, mcp |
| **Providers** | 20+ (OpenAI-compatible dominant) |
| **Channels** | 12 platforms (Telegram, Discord, Slack, Feishu, etc.) |
| **Notable Patterns** | Registry/plugin pattern, concurrent tool execution, token-based memory |
| **Stars** | 36,324 |
| **License** | MIT (inferred from project structure) |
| **CI** | GitHub Actions - Python 3.11, 3.12, 3.13 |
