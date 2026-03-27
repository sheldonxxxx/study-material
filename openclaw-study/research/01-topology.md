# OpenClaw Project Topology

**Repo:** `/Users/sheldon/Documents/claw/reference/openclaw`
**Date:** 2026-03-26
**Version:** 2026.3.24

---

## 1. Root Entry Points

### CLI Entry (Primary)
- **`openclaw.mjs`** (root, executable, shebang `#!/usr/bin/env node`)
  - Node.js version guard (requires Node 22.12.0+)
  - Enables module compile cache
  - Handles `--help` fast-path with precomputed root help text
  - Tries to import `./dist/entry.js` or `./dist/entry.mjs`
  - Falls back to source `./src/entry.ts` if dist missing (with error guidance)

### Package.json Bin
```json
"bin": { "openclaw": "openclaw.mjs" }
```

### Main Library Export
- **`dist/index.js`** (built from `src/index.ts`)
- **`src/index.ts`** - exports library functions + runs CLI when executed as main
- **`src/library.ts`** - core library exports (session management, config, port handling)

### Workspace Entry
```yaml
packages:
  - .              # root (openclaw CLI + core)
  - ui             # web UI package
  - packages/*     # internal packages (clawdbot, moltbot)
  - extensions/*   # plugin extensions
```

---

## 2. Monorepo Structure

```
openclaw/
├── .github/            # GitHub workflows, PR templates, labeler config
├── .vscode/            # VSCode settings
├── apps/               # Native mobile/desktop applications
│   ├── android/        # Android (Kotlin, Gradle)
│   ├── ios/            # iOS (Swift, XcodeGen)
│   ├── macos/          # macOS (Swift, XcodeGen)
│   └── shared/         # Shared OpenClawKit
├── assets/             # Static assets (icons, images)
├── docs/               # Mintlify documentation (44 subdirs)
├── extensions/          # Plugin extensions (85 packages)
├── packages/            # Internal packages (2)
│   ├── clawdbot/       # Discord bot
│   └── moltbot/        # Molt integration bot
├── patches/            # pnpm patches for dependencies
├── scripts/            # Build/dev/admin scripts (172 files)
├── skills/             # Agent skills (53 packages)
├── src/                 # Core TypeScript source
├── Swabble/            # Swift sub-project (separate package)
├── test/               # E2E tests, fixtures, helpers
├── test-fixtures/      # Test fixtures
├── ui/                 # Web UI (Lit-based components)
├── vendor/             # Vendored dependencies
├── git-hooks/          # Git pre-commit hooks
├── CLAUDE.md → AGENTS.md
├── Dockerfile          # Production Docker image
├── Dockerfile.sandbox*  # Sandbox Docker variants
├── docker-compose.yml
├── package.json        # Root package (pnpm workspace)
├── pnpm-lock.yaml
├── pnpm-workspace.yaml
├── tsconfig.json
├── tsdown.config.ts    # Build configuration
├── vitest.config.ts    # Test configuration
└── openclaw.mjs        # CLI entry point (root)
```

---

## 3. Source Organization (`src/`)

```
src/ (84 directories, core TypeScript)
├── index.ts             # Library entry + CLI runner
├── library.ts           # Core library exports
├── entry.ts             # CLI main dispatcher
├── entry.respawn.ts     # CLI respawn logic
├── version.ts           # Version handling
├── runtime.ts          # Runtime constants
├── logger.ts           # Logging setup
├── utils.ts             # Shared utilities

Core subsystems:
├── cli/                 # CLI commands (196 files)
│   ├── run-main.ts     # Main CLI runner
│   ├── program/        # Commander.js root program
│   ├── gateway-cli/   # Gateway subcommands
│   ├── plugins-cli/   # Plugin management
│   ├── config-cli.ts  # Config management
│   ├── secrets-cli.ts # Secrets management
│   └── ...            # 30+ command files

├── gateway/            # Gateway server (284 files)
│   ├── boot.ts         # Gateway startup
│   ├── client.ts       # Gateway client
│   ├── call.ts         # Call handling
│   ├── auth.ts         # Authentication
│   ├── agent.ts        # Agent logic
│   └── channels/       # Channel implementations

├── commands/           # Command bundle/manifest (189 files)
│   ├── bundle-manifest.ts
│   ├── bundle-commands.ts
│   └── bundled-plugin-metadata.generated.ts

├── plugins/            # Plugin SDK (189 files)
│   └── plugin-sdk/     # Plugin development kit

├── config/             # Configuration (236 files)

├── channels/           # Channel integrations (69 files)

├── agents/             # Agent system (642 files)
│   ├── agent.ts
│   ├── agents.commands.*.ts
│   └── ...

├── infra/              # Infrastructure (432 files)
│   ├── env.ts
│   ├── is-main.ts
│   └── ...

├── memory/             # Memory system (108 files)

├── media/              # Media processing (50 files)
├── media-understanding/ # Media understanding (58 files)
├── image-generation/    # Image generation (11 files)

├── tui/                # Terminal UI (34 files)

├── terminal/           # Terminal utilities (21 files)

├── security/           # Security (40 files)

├── secrets/            # Secrets management (55 files)

├── logging/            # Logging (33 files)

├── hooks/              # Hooks system (45 files)

├── daemon/             # Daemon (58 files)

├── cron/               # Cron jobs (79 files)

├── browser/            # Browser automation (139 files)

├── acp/                # ACP protocol (34 files)

├── auto-reply/         # Auto-reply (75 files)

├── node-host/          # Node host (18 files)

├── pairing/            # Device pairing (11 files)

├── sessions/           # Session management (17 files)

├── test-utils/         # Test utilities (37 files)

├── plugins/             # Plugin system (189 files)

├── shared/              # Shared utilities (68 files)

├── utils/               # Utilities (32 files)

├── routing/             # Routing (13 files)

├── i18n/                # Internationalization (3 files)

├── interactive/         # Interactive prompts (4 files)

├── line/                # LINE messaging (3 files)

├── link-understanding/  # Link understanding (9 files)

├── context-engine/      # Context engine (9 files)

├── canvas-host/         # Canvas host (8 files)

├── types/               # Type definitions (13 files)

├── process/             # Process management (19 files)

├── test-helpers/        # Test helpers (10 files)

├── web-search/          # Web search (4 files)

├── wizard/              # Setup wizard (18 files)

├── tts/                 # Text-to-speech (14 files)

├── compat/              # Compatibility (3 files)

├── docs/                # Inline docs (3 files)

└── scripts/             # Build scripts (5 files)
```

### Key Patterns in `src/`
- Tests colocated: `*.test.ts` next to source
- Runtime boundary: `*.runtime.ts` for lazy-loaded code
- Generated files: `*.generated.ts` (metadata, bundles)
- Channel code: `src/channels/` for built-in channels
- Plugin code: `extensions/<id>/` for external plugins

---

## 4. Extensions (`extensions/`)

**85 extension packages** - plugin-based channel providers and integrations.

### Sample Extensions:
```
extensions/
├── acpx/                 # ACP protocol extension
├── amazon-bedrock/       # AWS Bedrock provider
├── anthropic/            # Anthropic provider
├── brave/                # Brave search
├── byteplus/             # BytePlus provider
├── chutes/               # Chutes provider
├── deepgram/             # Deepgram speech
├── deepseek/             # DeepSeek provider
├── device-pair/          # Device pairing
├── discord/              # Discord channel
├── duckduckgo/           # DuckDuckGo search
├── exa/                  # Exa search
├── fal/                  # Fal.ai
├── feishu/               # Feishu (Lark)
├── firecrawl/            # Web scraping
├── github-copilot/       # GitHub Copilot
├── google/               # Google AI provider
├── googlechat/           # Google Chat
├── groq/                 # Groq provider
├── imessage/             # iMessage
├── irc/                  # IRC
├── kimi-coding/          # Kimi coding
├── line/                 # LINE messaging
├── lobster/              # Lobster provider
├── matrix/               # Matrix protocol
├── mattermost/           # Mattermost
├── memory-lancedb/        # LanceDB memory
├── microsoft/            # Microsoft provider
├── minimax/              # MiniMax provider
├── moonshot/             # Moonshot AI
├── msteams/              # Microsoft Teams
├── nextcloud-talk/       # Nextcloud Talk
├── openai/               # OpenAI provider
├── opencode/             # OpenCode
├── openrouter/           # OpenRouter
├── perplexity/           # Perplexity
├── phone-control/        # Phone control
├── qianfan/              # Qianfan provider
├── qwen-portal-auth/     # Qwen auth
├── signal/              # Signal
├── slack/                # Slack
├── swabble/              # Swabble extension
├── telegram/             # Telegram
├── tlon/                 # Tlon
├── together/             # Together AI
├── twitch/               # Twitch
├── twitter/              # Twitter/X
├── volcengine/           # Volcengine
├── whatsapp/             # WhatsApp
├── xai/                  # xAI
├── zalo/                 # Zalo
├── zalouser/             # Zalo user
└── shared/               # Shared extension utilities
```

### Extension Structure
Each extension follows pattern:
```
extensions/<name>/
├── package.json
├── src/
│   ├── api.ts           # Public API (extension -> openclaw)
│   ├── runtime-api.ts   # Runtime API
│   └── *.ts             # Implementation
└── openclaw.plugin.json  # Plugin manifest
```

---

## 5. Skills (`skills/`)

**53 skill packages** for agent capabilities.

```
skills/
├── 1password/
├── apple-notes/
├── apple-reminders/
├── bear-notes/
├── blogwatcher/
├── blucli/
├── bluebubbles/
├── camsnap/
├── canvas/
├── clawhub/
├── coding-agent/
├── discord/
├── eightctl/
├── gemini/
├── gh-issues/
├── gifgrep/
├── github/
├── gog/
├── goplaces/
├── healthcheck/
├── himalaya/
├── imsg/
├── mcporter/
├── model-usage/
├── nano-pdf/
├── node-connect/
├── notion/
├── obsidian/
├── openai-whisper/
├── openai-whisper-api/
├── openhue/
├── oracle/
├── ordercli/
├── peekaboo/
├── sag/
├── session-logs/
├── sherpa-onnx-tts/
├── skill-creator/
├── slack/
├── songsee/
├── sonoscli/
├── spotify-player/
├── summarize/
├── things-mac/
├── tmux/
├── trello/
├── video-frames/
├── voice-call/
├── wacli/
├── weather/
└── xurl/
```

---

## 6. Apps (`apps/`)

**Native mobile/desktop applications.**

```
apps/
├── android/              # Android app (Kotlin, Gradle)
│   ├── app/             # Main app source
│   ├── benchmark/        # Benchmark app
│   ├── gradle/          # Gradle wrapper
│   └── scripts/          # Build scripts
├── ios/                  # iOS app (Swift, XcodeGen)
│   └── Sources/          # Swift sources
├── macos/                # macOS app (Swift, XcodeGen)
│   └── Sources/          # Swift sources
└── shared/               # OpenClawKit (shared Swift package)
    └── Sources/
        ├── SwabbleCore/ # Core Swift
        ├── SwabbleKit/  # Shared kit
        └── swabble/     # Main swabble module
```

---

## 7. Swabble Sub-Project

**Separate Swift package** in `Swabble/` directory.

```
Swabble/
├── Package.swift        # Swift package manifest
├── Package.resolved      # Resolved dependencies
├── README.md
├── CHANGELOG.md
├── LICENSE
├── Sources/
│   ├── SwabbleCore/     # Core functionality
│   ├── SwabbleKit/      # Shared kit
│   └── swabble/         # Main module
├── Tests/               # Swift tests
├── docs/               # Documentation
├── scripts/            # Build scripts
├── .github/            # GitHub config
├── .swiftformat
└── .swiftlint.yml
```

---

## 8. UI (`ui/`)

**Web UI package** (Lit-based components).

```
ui/
├── public/              # Static public assets
├── src/
│   ├── main.ts         # Entry point
│   ├── styles.css      # Global styles
│   ├── local-storage.ts
│   ├── i18n/           # Internationalization
│   ├── styles/         # CSS styles
│   │   └── *.css
│   ├── types/          # Type definitions
│   └── ui/             # Lit components (79 files)
│       ├── components/
│       └── ...
└── package.json
```

---

## 9. Scripts (`scripts/`)

**172 build/dev/admin scripts** including:
- `tsdown-build.mjs` - TypeScript build
- `runtime-postbuild.mjs` - Runtime post-processing
- `test-parallel.mjs` - Test runner
- `run-node.mjs` - Development runner
- `protocol-gen.ts` - Protocol code generation
- Docker E2E tests
- Release workflows
- iOS/Android build scripts

---

## 10. Tests (`test/`)

```
test/
├── fixtures/           # Test fixtures
├── helpers/            # Test helpers
├── mocks/              # Mock implementations
├── scripts/            # Test scripts
├── e2e/                # E2E tests (*.e2e.test.ts)
└── ...
```

**Vitest configs:**
- `vitest.config.ts` - Default
- `vitest.unit.config.ts` - Unit tests
- `vitest.e2e.config.ts` - E2E tests
- `vitest.gateway.config.ts` - Gateway tests
- `vitest.live.config.ts` - Live/integration tests
- `vitest.performance-config.ts` - Performance tests

---

## 11. Key Dependencies

**Runtime:**
- `hono` (4.12.8) - Web framework
- `express` (5.2.1) - HTTP server
- `ws` (8.20.0) - WebSocket
- `zod` (4.3.6) - Schema validation
- `@sinclair/typebox` (0.34.48) - Type schemas
- `yaml` (2.8.3) - YAML parsing
- `dotenv` (17.3.1) - Environment config
- `commander` (14.0.3) - CLI
- `@clack/prompts` (1.1.0) - TTY prompts
- `chokidar` (5.0.0) - File watching
- `sharp` (0.34.5) - Image processing
- `playwright-core` (1.58.2) - Browser automation
- `pdfjs-dist` (5.5.207) - PDF processing

**AI/Agent:**
- `@mariozechner/pi-agent-core` (0.61.1)
- `@mariozechner/pi-ai` (0.61.1)
- `@mariozechner/pi-coding-agent` (0.61.1)
- `@mariozechner/pi-tui` (0.61.1)
- `@modelcontextprotocol/sdk` (1.27.1)

**Dev:**
- `typescript` (5.9.3)
- `vitest` (4.1.0)
- `tsx` (4.21.0)
- `tsdown` (0.21.4)
- `oxlint` / `oxfmt` - Linting/formatting

---

## 12. Build System

- **Package Manager:** pnpm (10.32.1)
- **Build Tool:** tsdown (for TypeScript bundling)
- **Runtime:** Node 22.14.0+ required
- **Also supports:** Bun for dev/scripts

**Build output:**
- `dist/` - Compiled JavaScript
- `dist/plugin-sdk/` - Plugin SDK types
- `dist/cli/` - CLI commands
- `dist/gateway/` - Gateway server

---

## 13. CLI Commands

Key command modules in `src/cli/`:
- `gateway-cli/` - Gateway management
- `plugins-cli/` - Plugin lifecycle
- `config-cli.ts` - Configuration
- `secrets-cli.ts` - Secrets
- `channels-cli.ts` - Channel management
- `daemon-cli/` - Daemon control
- `nodes-cli/` - Node management
- `cron-cli/` - Cron jobs
- `memory-cli.runtime.ts` - Memory commands
- `pairing-cli.ts` - Device pairing
- `qr-cli.ts` - QR code generation
- `update-cli/` - Auto-update
- `completion-cli.ts` - Shell completions
- `mcp-cli.ts` - MCP tools
- `hooks-cli.ts` - Hook management
- `logs-cli.ts` - Log retrieval
- `browser-cli/` - Browser automation

---

## 14. Configuration

**Config schema:** Generated via `pnpm config:schema:gen`
**Config docs:** Generated via `pnpm config:docs:gen`

Key config locations:
- `~/.openclaw/` - User config directory
- `.env.example` - Environment template
- `openclaw.podman.env` - Podman environment

---

## Summary

OpenClaw is a **multi-channel AI gateway** with:

1. **Monorepo structure** - pnpm workspace with root + ui + packages + extensions
2. **Core CLI** in `src/cli/` (Commander.js-based)
3. **Gateway server** in `src/gateway/` (Hono/Express)
4. **85 extension plugins** in `extensions/` for channels/providers
5. **53 agent skills** in `skills/` for capabilities
6. **Native apps** in `apps/` (iOS, Android, macOS)
7. **Web UI** in `ui/` (Lit components)
8. **Swabble** - Separate Swift sub-project
9. **Comprehensive test suite** with Vitest
10. **Plugin SDK** for extension development

**Entry flow:**
```
openclaw.mjs → dist/entry.js → src/entry.ts → src/cli/run-main.ts → commands
```
