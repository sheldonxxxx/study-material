# PicoClaw Project Topology

## Overview

**Project:** PicoClaw - Ultra-lightweight personal AI agent
**Module:** `github.com/sipeed/picoclaw`
**Language:** Go (v1.25.8)
**Type:** Monorepo (single Go module with embedded frontend)

## Project Type

Single Go module (`go.mod`) with:
- CLI application (main agent)
- Web server (embedded frontend + API)
- TUI launcher application
- Example projects
- All in one git repository

## Directory Structure

```
picoclaw/
├── cmd/                          # Command entry points
│   ├── picoclaw/                 # Main CLI agent
│   │   ├── main.go               # Entry point
│   │   └── internal/             # Command-specific subcommands
│   │       ├── onboard/
│   │       ├── agent/
│   │       ├── auth/
│   │       ├── gateway/
│   │       ├── status/
│   │       ├── cron/
│   │       ├── migrate/
│   │       ├── skills/
│   │       ├── model/
│   │       └── version/
│   └── picoclaw-launcher-tui/    # TUI configuration launcher
│       ├── main.go               # Entry point
│       ├── ui/                   # TUI components
│       └── config/               # Launcher config
│
├── pkg/                          # Shared libraries (core logic)
│   ├── agent/                    # AI agent core
│   │   ├── loop.go               # Main agent loop (107KB - largest file)
│   │   ├── context.go            # Context management
│   │   ├── steering.go           # Model steering
│   │   ├── turn.go / subturn.go  # Turn management
│   │   ├── instance.go           # Agent instance
│   │   └── *.go
│   ├── channels/                 # Chat platform integrations
│   │   ├── manager.go            # Channel manager
│   │   ├── telegram/
│   │   ├── discord/
│   │   ├── slack/
│   │   ├── matrix/
│   │   ├── feishu/
│   │   ├── dingtalk/
│   │   ├── wecom/
│   │   ├── weixin/
│   │   ├── qq/
│   │   ├── line/
│   │   ├── whatsapp/
│   │   ├── whatsapp_native/
│   │   ├── irc/
│   │   ├── onebot/
│   │   ├── maixcam/
│   │   └── pico/                 # PicoClaw's own channel
│   ├── providers/                # AI model providers
│   │   ├── anthropic/
│   │   ├── openai/
│   │   ├── aws/                  # Bedrock
│   │   └── ...                   # 42 subdirectories total
│   ├── commands/                 # Command handling
│   ├── config/                   # Configuration management
│   ├── auth/                    # Authentication
│   ├── bus/                     # Event bus
│   ├── skills/                  # Skill system
│   ├── tools/                   # Tools registry (55 files)
│   ├── memory/                  # Memory system
│   ├── voice/                   # Voice processing
│   ├── mcp/                     # Model Context Protocol
│   ├── routing/                 # Message routing
│   ├── session/                 # Session management
│   ├── gateway/                 # Gateway functionality
│   ├── media/                   # Media handling
│   └── [other util packages]
│
├── web/                         # Web interface
│   ├── backend/                 # Go web server
│   │   ├── main.go              # Entry point (http server on :18800)
│   │   ├── api/                 # REST API handlers
│   │   ├── middleware/          # HTTP middleware
│   │   ├── model/               # API models
│   │   ├── utils/               # Backend utilities
│   │   ├── launcherconfig/      # Launcher config handling
│   │   └── dist/                # Embedded frontend build
│   └── frontend/                # React frontend
│       ├── src/                  # Frontend source
│       │   ├── main.tsx         # React entry point
│       │   ├── routes/          # Page routes
│       │   ├── components/      # Reusable components
│       │   ├── features/        # Feature modules
│       │   ├── hooks/           # React hooks
│       │   ├── store/           # State management
│       │   └── lib/             # Frontend utilities
│       └── public/              # Static assets
│
├── workspace/                    # Agent workspace
│   ├── memory/                   # Memory storage
│   └── skills/                  # Agent skills
│       ├── agent-browser/
│       ├── github/
│       ├── tmux/
│       ├── weather/
│       └── skill-creator/
│
├── config/                      # Configuration files
├── assets/                      # Static assets (icons, images)
├── docker/                      # Docker-related files
├── docs/                        # Documentation
│   ├── design/
│   ├── hooks/
│   ├── channels/               # Channel-specific docs
│   │   ├── telegram/
│   │   ├── discord/
│   │   ├── slack/
│   │   └── [other channels]
│   └── [i18n docs: fr, ja, zh, it, pt-br, vi]
│
├── examples/                    # Example projects
│   └── pico-echo-server/
│
├── scripts/                     # Build/utility scripts
│
└── [root files]
    ├── go.mod / go.sum          # Go dependencies
    ├── Makefile                 # Build automation
    ├── README.md                # Main documentation
    └── .goreleaser.yaml         # Release configuration
```

## Entry Points

### 1. Main CLI Agent
**File:** `cmd/picoclaw/main.go`

```
picoclaw [command]

Commands:
  onboard    - Initial setup/onboarding
  agent      - Run the AI agent
  auth       - Authentication management
  gateway    - Gateway control
  status     - Show status
  cron       - Scheduled tasks
  migrate    - Migration tools
  skills     - Skill management
  model      - Model configuration
  version    - Show version
```

### 2. Web Console Server
**File:** `web/backend/main.go`

```
PicoClaw Launcher - Web-based configuration editor
Listens on: http://localhost:18800 (default)

Flags:
  -port int         Port to listen on (default 18800)
  -public           Listen on all interfaces (0.0.0.0)
  -no-browser       Don't auto-open browser
  -lang string      Language (en/zh)
  -console          Console mode (no GUI)

Provides:
  - Web UI at /
  - REST API at /api/*
  - Embedded frontend assets
```

### 3. TUI Launcher
**File:** `cmd/picoclaw-launcher-tui/main.go`

```
Terminal-based configuration UI
```

## Key Packages (pkg/)

| Package | Purpose |
|---------|---------|
| `pkg/agent` | Core AI agent logic, loop, context management |
| `pkg/channels` | Multi-platform chat integrations (telegram, discord, etc.) |
| `pkg/providers` | AI model provider integrations (Anthropic, OpenAI, AWS, etc.) |
| `pkg/commands` | Command parsing and execution |
| `pkg/skills` | Skill system for extending capabilities |
| `pkg/tools` | Tool registry and implementations |
| `pkg/memory` | Memory/persistence layer |
| `pkg/gateway` | Gateway for channel connections |
| `pkg/session` | Session management |

## Web Stack

### Backend
- **Language:** Go
- **HTTP:** Standard library `net/http`
- **Framework:** Custom (no framework visible)
- **Port:** 18800 (default)

### Frontend
- **Framework:** React
- **Build:** Vite
- **Package Manager:** pnpm
- **Styling:** Tailwind CSS (components.json references shadcn/ui)
- **Router:** TanStack Router (routeTree.gen.ts)
- **State:** Zustand or similar (store/ directory)

## Build System

- **Go Build:** Standard `go build` via Makefile
- **Releaser:** goreleaser (`.goreleaser.yaml`)
- **Linter:** golangci-lint (`.golangci.yaml`)
- **Frontend:** Vite + TypeScript

## Dependencies

Single Go module with major dependencies:
- CLI: `spf13/cobra` (commands)
- AI: `anthropics/anthropic-sdk-go`, `openai/openai-go`
- Channels: `discordgo`, `slack-go/slack`, `telego` (Telegram), `mautrix` (Matrix)
- Web: `gorilla/websocket`
- Database: `modernc.org/sqlite`
- Logging: `rs/zerolog`
- Config: `BurntSushi/toml`

## Key Insights

1. **Modular Architecture:** Clean separation between core agent (`pkg/agent`), channels (`pkg/channels`), and providers (`pkg/providers`)

2. **Multi-Platform:** Supports 15+ chat platforms (Telegram, Discord, Slack, Matrix, WhatsApp, etc.)

3. **Provider Agnostic:** AI models abstracted through providers package, supporting Anthropic, OpenAI, AWS Bedrock

4. **Three Entry Points:**
   - CLI agent (`picoclaw`)
   - Web console (`picoclaw-web`)
   - TUI launcher (`picoclaw-launcher-tui`)

5. **Embedded Frontend:** Web UI is compiled and embedded into the web backend binary

6. **Channel Protocol:** PicoClaw has its own "Pico Channel" (websocket-based) for communication between web UI and agent
