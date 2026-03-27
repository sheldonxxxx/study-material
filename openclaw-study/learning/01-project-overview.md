# OpenClaw — Project Overview

**Version:** 2026.3.24
**License:** MIT
**Repository:** https://github.com/openclaw/openclaw

---

## Project Description

OpenClaw is a multi-channel AI gateway with extensible messaging integrations — "the AI that actually does things." It runs on user devices, in their channels, with their rules.

The core value proposition is a **hub-and-spoke architecture** where a local gateway server acts as the control plane, connecting to 20+ messaging platforms simultaneously and providing a unified AI assistant experience across all channels.

### Tagline
"The AI that actually does things."

---

## Directory Structure Overview

```
openclaw/
├── .github/            # GitHub workflows, PR templates, CODEOWNERS
├── .vscode/            # VSCode settings
├── apps/               # Native mobile/desktop applications
│   ├── android/        # Android (Kotlin, Gradle)
│   ├── ios/            # iOS (Swift, XcodeGen)
│   ├── macos/          # macOS (Swift, XcodeGen)
│   └── shared/         # Shared OpenClawKit Swift package
├── assets/             # Static assets (icons, images)
├── docs/               # Mintlify documentation (44 subdirs)
├── extensions/         # Plugin extensions (85 packages)
├── packages/           # Internal packages (clawdbot, moltbot)
├── patches/            # pnpm patches for dependencies
├── scripts/            # Build/dev/admin scripts (172 files)
├── skills/             # Agent skills (53 packages)
├── src/                # Core TypeScript source (84 directories)
├── Swabble/            # Standalone Swift sub-project
├── test/               # E2E tests, fixtures, helpers
├── test-fixtures/      # Test fixtures
├── ui/                 # Web UI (Lit-based components)
├── vendor/             # Vendored dependencies
├── CLAUDE.md → AGENTS.md
├── Dockerfile          # Production Docker image
├── package.json        # Root package (pnpm workspace)
├── pnpm-workspace.yaml
├── tsconfig.json
├── tsdown.config.ts
├── vitest.config.ts
└── openclaw.mjs        # CLI entry point
```

---

## Monorepo Layout

### Workspace Configuration (pnpm-workspace.yaml)

```yaml
packages:
  - .              # Root: CLI, gateway, core
  - ui             # Control UI (web interface)
  - packages/*     # Internal packages (clawdbot, moltbot)
  - extensions/*   # Plugin extensions
```

### Packages (`packages/`)

| Package | Purpose |
|---------|---------|
| `clawdbot` | Discord bot integration |
| `moltbot` | Molt integration bot |

### Extensions (`extensions/`)

85 plugin packages providing channel integrations and provider connections. Key examples:

**Messaging Channels:**
- Discord, Slack, Telegram, WhatsApp, Signal
- iMessage (BlueBubbles), Matrix, Microsoft Teams, LINE
- IRC, Mattermost, Nextcloud Talk, Nostr, Twitch, Twitter

**AI Providers:**
- OpenAI, Anthropic, Google AI, Azure OpenAI, AWS Bedrock
- DeepSeek, Groq, Moonshot, Ollama, LM Studio, vLLM

**Services:**
- Brave Search, DuckDuckGo, Exa, Firecrawl (web scraping)
- ElevenLabs (TTS), Deepgram (STT)
- GitHub Copilot, MCP (mcporter bridge)

### Skills (`skills/`)

53 skill packages for agent capabilities:

```
skills/
├── 1password/          # 1Password integration
├── apple-notes/        # Apple Notes access
├── discord/            # Discord operations
├── github/             # GitHub integration
├── notion/             # Notion integration
├── obsidian/           # Obsidian vault access
├── spotify-player/     # Spotify control
├── summarize/          # Content summarization
├── weather/            # Weather information
└── ... (43 more)
```

### Apps (`apps/`)

Native applications for desktop and mobile:

| Platform | Language | Build System |
|----------|----------|--------------|
| macOS | Swift | XcodeGen + SPM |
| iOS | Swift | XcodeGen + SPM |
| Android | Kotlin | Gradle |

### UI (`ui/`)

Web-based control interface using Lit web components:

```
ui/
├── public/
├── src/
│   ├── main.ts
│   ├── styles.css
│   ├── i18n/
│   ├── styles/
│   ├── types/
│   └── ui/              # 79 Lit component files
└── package.json
```

### Source (`src/`)

84 directories of core TypeScript code organized by subsystem:

| Subsystem | Files | Purpose |
|-----------|-------|---------|
| `agents/` | 642 | Pi agent runtime, command handling |
| `infra/` | 432 | Infrastructure utilities |
| `gateway/` | 284 | Gateway server implementation |
| `cli/` | 196 | CLI commands (Commander.js) |
| `commands/` | 189 | Command bundle/manifest |
| `plugins/` | 189 | Plugin SDK |
| `config/` | 236 | Configuration management |
| `browser/` | 139 | Browser automation (CDP) |
| `memory/` | 108 | Memory system |
| `media/` | 50 | Media processing |
| `media-understanding/` | 58 | Image/audio understanding |
| `cron/` | 79 | Scheduled jobs |
| `acp/` | 34 | ACP protocol |

---

## Key Entry Points

### CLI Entry (Primary)

**`openclaw.mjs`** (root, executable):
- Node.js version guard (requires Node 22.12.0+)
- Enables module compile cache
- Handles `--help` fast-path
- Imports `./dist/entry.js` or falls back to `./src/entry.ts`

```bash
# Via bin
openclaw --help

# Via npx
npx openclaw --help
```

### Entry Flow

```
openclaw.mjs
  → dist/entry.js (or src/entry.ts fallback)
    → src/cli/run-main.ts
      → Commander.js commands
```

### Library Export

- **`dist/index.js`** — Built from `src/index.ts`
- **`src/index.ts`** — Exports library functions + runs CLI when executed as main
- **`src/library.ts`** — Core library exports (session management, config, port handling)

### Gateway Server

- **Default URL:** `ws://127.0.0.1:18789`
- **Built with:** Express 5 + Hono 4
- **WebSocket:** `ws` package for real-time communication

---

## Community and Governance Summary

### Maintainer Team (22 members)

Led by **Peter Steinberger** (Benevolent Dictator, steipete@gmail.com) with specialized teams:

| Team | Focus |
|------|-------|
| Core | Peter, Josh Avant, Jonathan Taylor |
| Agents | Vincent Koc, Tyler Yust, Gustavo Madeira |
| Mobile (iOS) | Mariano Belinky, Nimrod Gutman |
| Mobile (Android) | Ayaan Zaidi |
| Desktop (macOS) | Tyler Yust, Nimrod Gutman |
| Security | Robin Waslander, Mariano Belinky, Vincent Koc |
| Docs/UX | Val Alexander, Seb Slight, Radek Sienkiewicz |
| Channels | Shadow (Discord), Jos (Telegram), Vignesh (IRC) |

### Communication Channels

- **Discord:** https://discord.gg/qkhbAGHRBT (active community, help channels)
- **GitHub:** Issues, PRs, Discussions
- **X/Twitter:** @steipete, @openclaw
- **Security:** security@openclaw.ai

### Release Cadence

- **Versioning:** `YYYY.M.D` format (e.g., 2026.3.24)
- **Cadence:** Multiple stable releases per week with beta previews
- **Changelog:** Grouped by Breaking, Changes, Fixes with PR attribution

### Contribution Policy

- Bugs/small fixes: Open PR directly
- Architecture changes: Start with GitHub Discussion or Discord
- Refactor-only PRs: Not accepted unless explicitly requested
- AI/Vibe-coded PRs: Explicitly welcomed and treated as first-class
- PR template requires: Root cause analysis, regression test plan, human verification

---

## Feature Summary

### Core Features (Tier 1)

| Feature | Description | Key Files |
|---------|-------------|-----------|
| Multi-Channel Hub | 20+ messaging platforms via extension plugins | `extensions/*/`, `src/channels/` |
| Gateway Control Plane | Local WebSocket server on port 18789 | `src/gateway/`, `src/infra/` |
| Pi Agent Runtime | AI processing with tool/stream support | `src/agents/`, `src/context-engine/` |
| Voice Wake + Talk | Voice activation on macOS/iOS/Android | `src/tts/`, `extensions/elevenlabs/` |
| Live Canvas | Visual agent workspace with A2UI protocol | `src/canvas-host/`, `apps/*/Sources/Canvas/` |
| Browser Control | CDP-controlled Chrome/Chromium | `src/browser/` |
| CLI Interface | Full-featured command-line tool | `src/cli/`, `openclaw.mjs` |

### Secondary Features (Tier 2)

| Feature | Description | Key Files |
|---------|-------------|-----------|
| Device Nodes | macOS/iOS/Android companion apps | `apps/*/`, `src/node-host/` |
| Skills Platform | Extensible capability packages | `skills/`, `src/commands/skills-cli.*` |
| Security & Sandboxing | Docker isolation, secrets management | `src/security/`, `src/secrets/` |
| Media Pipeline | Image/audio/video processing | `src/media/`, `src/media-understanding/` |
| Automation | Cron jobs, webhooks, MCP integration | `src/cron/`, `extensions/mcporter/` |

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Messaging Channels                        │
│  WhatsApp | Telegram | Slack | Discord | Signal | iMessage  │
│  Matrix | MS Teams | LINE | + 13 more                        │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      OpenClaw Gateway                        │
│              (ws://127.0.0.1:18789)                         │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐             │
│  │ Sessions│ │ Channels│ │  Tools  │ │  Cron   │             │
│  │         │ │         │ │ Browser │ │ Webhooks│             │
│  │         │ │         │ │ Canvas  │ │  MCP    │             │
│  └────┬────┘ └────┬────┘ └────┬────┘ └─────────┘             │
│       │           │           │                               │
│       └───────────┴───────────┘                               │
│                      │                                         │
│               ┌──────▼──────┐                                 │
│               │  Pi Agent   │                                 │
│               │  (Runtime)  │                                 │
│               └─────────────┘                                 │
└─────────────────────────────┬───────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│   macOS App   │     │   iOS Node    │     │ Android Node  │
│  Menu bar UI  │     │   Canvas      │     │  Chat/Voice   │
│  Voice Wake   │     │   Voice Wake  │     │  Camera/Screen│
│  WebChat      │     │   Camera      │     │  Device Cmds  │
└───────────────┘     └───────────────┘     └───────────────┘
```

---

## Key Technical Decisions

| Decision | Rationale |
|----------|-----------|
| ESM-only | Modern JavaScript standard, better tree-shaking |
| TypeScript strict mode | Catch errors at compile time |
| pnpm workspaces | Efficient monorepo dependency management |
| tsdown bundler | Fast TypeScript compilation |
| Vitest forks pool | Avoids `threads`/`vmForks` instability |
| Plugin SDK boundary | Extensions cannot import core internals |
| Local-first gateway | Privacy, no cloud dependency |
| Dynamic CI matrix | Only test affected subsystems |
