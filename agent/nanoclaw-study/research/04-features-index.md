# NanoClaw Feature Index

## Project Overview

NanoClaw is a lightweight AI assistant framework that runs Claude agents securely in isolated containers. The codebase is intentionally small (single Node.js process, ~15 source files) enabling users to fully understand and customize it.

---

## Core Features (Priority Tier 1)

| # | Feature | Description | Key Files |
|---|---------|-------------|-----------|
| 1 | **Container Isolation** | Agents run in isolated Linux containers (Docker on macOS/Linux, Apple Container on macOS) with filesystem isolation. Only explicitly mounted directories are accessible. | `src/container-runner.ts`, `src/container-runtime.ts`, `container/` |
| 2 | **Multi-Channel Messaging** | Unified interface for messaging via WhatsApp, Telegram, Discord, Slack, Gmail. Channels are added via skills (`/add-whatsapp`, `/add-telegram`, etc.) and self-register at startup. | `src/channels/registry.ts`, `src/index.ts`, `.claude/skills/add-*` |
| 3 | **Group-Based Context Isolation** | Each conversation group has its own `CLAUDE.md` memory, isolated filesystem mount, and runs in its own container sandbox. | `src/ipc.ts`, `src/group-queue.ts`, `groups/*/CLAUDE.md` |
| 4 | **Message Orchestration** | Single Node.js process polling loop that routes messages from all channels to Claude Agent SDK and back. | `src/index.ts` (main orchestrator), `src/router.ts` |
| 5 | **Scheduled Tasks** | Recurring jobs (cron-style) that run Claude automatically and can message results back. Configurable per-group. | `src/task-scheduler.ts`, `src/timezone.ts` |
| 6 | **Per-Group Queue with Concurrency Control** | Global concurrency limiting ensures resources are shared fairly across groups while preventing overload. | `src/group-queue.ts` |
| 7 | **Credential Security (Agent Vault)** | API keys and tokens never enter containers. Outbound requests route through OneCLI Agent Vault which injects credentials at request time with per-agent policies and rate limits. | `src/env.ts`, `setup/init-onecli/` |

---

## Secondary Features (Priority Tier 2)

| # | Feature | Description | Key Files |
|---|---------|-------------|-----------|
| 1 | **Web Access** | Agents can search and fetch content from the web for research tasks. | `container/skills/` (browser capability) |
| 2 | **Trigger Word Processing** | Messages are filtered by trigger word (default: `@Andy`) for targeted assistant interaction. | `src/config.ts` (trigger pattern), `src/sender-allowlist.ts` |
| 3 | **Main Channel (Self-Chat)** | Private admin channel for controlling groups, listing tasks, and managing the assistant without group context. | `groups/main/`, `src/ipc.ts` |
| 4 | **Agent Swarms** | Capability to spin up teams of specialized agents that collaborate on complex multi-step tasks. | `src/ipc.ts` (IPC primitives), `.claude/skills/add-telegram-swarm/` |
| 5 | **Sender Allowlist** | Security mechanism to restrict which senders can trigger the assistant in each group. | `src/sender-allowlist.ts` |
| 6 | **Channel Formatting** | Normalizes message formatting across different channel formats (Discord embeds, Telegram markdown, etc.) | `src/formatting.ts`, `.claude/skills/channel-formatting/` |
| 7 | **Database Persistence** | SQLite-based storage for messages, groups, sessions, and application state with migration support. | `src/db.ts`, `src/db-migration.test.ts` |

---

## Skill-Based Extensibility

NanoClaw uses a **skills-based architecture** rather than plugin system. Skills are Claude Code skill definitions that transform the fork. Four types:

| Skill Type | Purpose | Examples |
|------------|---------|----------|
| **Feature skills** | Add capabilities via code branches | `/add-whatsapp`, `/add-telegram`, `/add-discord` |
| **Utility skills** | Ship code files with SKILL.md | `/claw` (interactive mode) |
| **Operational skills** | Instruction-only workflows | `/setup`, `/debug`, `/customize` |
| **Container skills** | Loaded inside agent containers | `container/skills/` (browser, formatting) |

**Available skills in repo:**
- `add-compact`, `add-discord`, `add-emacs`, `add-gmail`, `add-image-vision`, `add-macos-statusbar`, `add-ollama-tool`, `add-parallel`, `add-pdf-reader`, `add-reactions`, `add-slack`, `add-telegram`, `add-telegram-swarm`, `add-voice-transcription`, `add-whatsapp`, `channel-formatting`, `claw`, `convert-to-apple-container`, `customize`, `debug`, `get-qodo-rules`, `init-onecli`, `qodo-pr-resolver`, `setup`, `update-nanoclaw`, `update-skills`, `use-local-whisper`, `use-native-credential-proxy`, `x-integration`

---

## Architecture Summary

```
Channels --> SQLite --> Polling loop --> Container (Claude Agent SDK) --> Response
     ^                                                                  |
     └───────────────────────── IPC via filesystem ─────────────────────┘
```

- **Single process**: One Node.js orchestrator
- **Self-registering channels**: Channel skills add themselves when credentials present
- **Per-group isolation**: Filesystem mount, CLAUDE.md memory, container sandbox per group
- **IPC**: Filesystem-based inter-process communication
- **Concurrency**: Global limit with per-group queuing

---

## Cross-Reference: README Claims vs Implementation

| README Claim | Implemented | Location |
|--------------|-------------|----------|
| Multi-channel messaging | Yes | `src/channels/registry.ts`, skill branches |
| Isolated group context | Yes | `groups/*/`, `src/ipc.ts` |
| Main channel | Yes | `groups/main/` |
| Scheduled tasks | Yes | `src/task-scheduler.ts` |
| Web access | Yes | `container/skills/` |
| Container isolation | Yes | `src/container-runner.ts`, `container/` |
| Credential security (Agent Vault) | Yes | `setup/init-onecli/`, `src/env.ts` |
| Agent Swarms | Yes | IPC primitives + swarm skill |
| Custom model endpoints | Yes | `ANTHROPIC_BASE_URL` env var support |

---

*Generated from README.md analysis and directory structure scan*
