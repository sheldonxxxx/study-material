# GitClaw — Learning Reference

## What This Is
Analysis of the GitClaw codebase for the purpose of informing development of a similar git-native AI agent framework.

## Key Takeaways
- **Git-native identity is the core insight**: Agents as repos, memory as commits, inheritance via git URLs — this is the project's defining architectural bet
- **Hexagonal architecture with pi-agent-core as domain center**: Well-structured with clear ports/adapters, but `execSync` blocking calls undermine async design
- **Voice UI is surprisingly large (1100+ lines)**: The most complex secondary feature; multi-provider (OpenAI Realtime, Gemini Live) with WhatsApp/Telegram integration
- **Critical gap: Workflows have no execution engine**: Feature 13 (Workflows) formats workflows for prompts but doesn't execute them — the agent interprets them itself
- **Learning system is sophisticated but underbuilt**: Asymmetric reinforcement learning (2x failure penalty), Jaccard similarity for skill novelty — but fields exist without auto-update
- **Silent failure is pervasive**: Empty catch blocks throughout; errors swallowed rather than surfaced — the most important code quality finding

## Should Trust / Should Not Trust
- **Trust**: Architecture decisions (hexagonal, plugin system, hook decorator), multi-model abstraction, skills/confidence learning system
- **Do Not Trust**: README claims about "simple" setup (code is complex); compliance enforcement (it's informational only); CI publishing (doesn't run tests)

## At a Glance
- **Language:** TypeScript 5.7, strict mode, ESM modules
- **Architecture:** Hexagonal (ports & adapters) with pi-agent-core as domain center
- **Key Libraries:** @mariozechner/pi-agent-core, @mariozechner/pi-ai, @sinclair/typebox, ws, baileys, node-cron
- **Notable Patterns:** Decorator (hooks), Factory (tools/sandbox), Strategy (voice adapters), Channel (SDK streaming), Repository (skills/plugins)
- **Stars / Activity:** ~100 commits, 3 weeks old, single maintainer, 5 contributors, MIT license
- **License:** MIT
