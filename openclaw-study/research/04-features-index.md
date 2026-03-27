# OpenClaw Feature Index

**Project:** OpenClaw - Personal AI Assistant
**Source:** README.md, VISION.md, src/, extensions/, apps/
**Date:** 2026-03-26

---

## Core Features (Priority Tier 1)

### 1. Multi-Channel Messaging Hub
**Description:** Unified inbox that connects to 20+ messaging platforms simultaneously, allowing users to interact with the AI assistant through their preferred communication channels.

**Channels Supported:**
- WhatsApp, Telegram, Slack, Discord, Google Chat
- Signal, iMessage, BlueBubbles (iMessage recommended)
- IRC, Microsoft Teams, Matrix, Feishu, LINE
- Mattermost, Nextcloud Talk, Nostr, Synology Chat
- Tlon, Twitch, Zalo, Zalo Personal, WeChat, WebChat

**Key Files/Locations:**
- `src/channels/` - Core channel implementations
- `extensions/discord/`, `extensions/slack/`, `extensions/telegram/`, `extensions/whatsapp/` - Channel plugins
- `extensions/msteams/`, `extensions/matrix/`, `extensions/line/`, `extensions/signal/` - Extended channels
- `docs/channels/` - Channel documentation

**Priority:** CORE

---

### 2. Local-First Gateway Control Plane
**Description:** Self-hosted WebSocket-based gateway that runs locally on user devices, serving as the central control plane for sessions, channels, tools, and events. The gateway stays on localhost by default with optional Tailscale exposure.

**Key Features:**
- WebSocket control plane (ws://127.0.0.1:18789)
- Session management with presence, typing indicators
- Configuration via `~/.openclaw/openclaw.json`
- Built-in web UI + WebChat
- Cron jobs and webhooks
- Tailscale Serve/Funnel for remote access

**Key Files/Locations:**
- `src/gateway/` - Gateway implementation
- `src/infra/` - Infrastructure components
- `src/cli/gateway-cli/` - Gateway CLI commands
- `ui/` - Web UI components
- `docs/gateway/` - Gateway documentation

**Priority:** CORE

---

### 3. AI Agent Runtime (Pi Agent)
**Description:** The core AI agent (codenamed Pi) that processes messages, executes tools, and coordinates responses across channels. Supports tool streaming, block streaming, multi-agent routing, and session isolation.

**Key Features:**
- RPC-based agent runtime
- Tool streaming and block streaming
- Multi-agent routing (route channels/accounts to isolated agents)
- Session model: main for direct chats, group isolation, activation modes
- Reply-back and queue modes
- Context compaction and session pruning

**Key Files/Locations:**
- `src/agents/` - Agent implementation (642 files)
- `src/agents/acp-spawn.ts` - Agent spawning logic
- `src/agents/agent-command.ts` - Agent command handling
- `src/context-engine/` - Context processing
- `docs/concepts/agent` - Agent documentation

**Priority:** CORE

---

### 4. Voice Wake + Talk Mode
**Description:** Voice interaction system that allows hands-free wake word activation on macOS/iOS and continuous voice input on Android. Uses ElevenLabs for voice synthesis with system TTS fallback.

**Key Features:**
- Voice Wake - wake word detection on macOS/iOS
- Talk Mode - continuous voice on Android
- Push-to-talk overlay (macOS)
- ElevenLabs voice synthesis
- System TTS fallback

**Key Files/Locations:**
- `src/tts/` - Text-to-speech implementation
- `extensions/elevenlabs/` - ElevenLabs integration
- `extensions/voice-call/` - Voice call extension
- `docs/nodes/voicewake`, `docs/nodes/talk` - Voice documentation

**Priority:** CORE

---

### 5. Live Canvas Visual Workspace
**Description:** Agent-driven visual workspace that renders a controllable canvas interface. Supports A2UI protocol for agent-to-UI communication, eval, snapshot, and push/reset operations.

**Key Features:**
- A2UI (Agent-to-UI) protocol
- Canvas push/reset, eval, snapshot
- Rendered in macOS/iOS/Android apps
- Browser-based canvas control

**Key Files/Locations:**
- `src/canvas-host/` - Canvas host implementation
- `src/canvas-host/a2ui/` - A2UI protocol
- `apps/macos/Sources/Canvas/` - macOS Canvas
- `docs/platforms/mac/canvas` - Canvas documentation

**Priority:** CORE

---

### 6. Browser Control
**Description:** Dedicated OpenClaw-managed Chrome/Chromium browser instance with CDP (Chrome DevTools Protocol) control for web navigation, screenshots, form filling, and web interactions.

**Key Features:**
- Openclaw-managed Chrome/Chromium
- CDP control for snapshots, actions, uploads
- Browser profiles support
- Screenshot capture

**Key Files/Locations:**
- `src/browser/` - Browser implementation (139 files)
- `src/cli/browser-cli-*` - Browser CLI commands
- `docs/tools/browser` - Browser documentation

**Priority:** CORE

---

### 7. Command-Line Interface (CLI)
**Description:** Full-featured CLI (`openclaw`) for gateway management, agent interaction, message sending, channel configuration, device pairing, plugin management, and more.

**Key Commands:**
- `openclaw onboard` - Step-by-step setup wizard
- `openclaw gateway` - Gateway management
- `openclaw agent` - Agent interaction
- `openclaw message send` - Direct messaging
- `openclaw channels` - Channel management
- `openclaw devices` - Device/node pairing
- `openclaw plugins` - Plugin installation/management
- `openclaw config` - Configuration management
- `openclaw doctor` - Troubleshooting diagnostics

**Key Files/Locations:**
- `src/cli/` - CLI implementation (196 files)
- `src/commands/` - Command implementations
- `openclaw.mjs` - CLI entry point
- `docs/cli/` - CLI documentation

**Priority:** CORE

---

## Secondary Features (Priority Tier 2)

### 8. Device Nodes (macOS/iOS/Android)
**Description:** Companion apps that pair with the Gateway to expose device-local capabilities. Each device runs as a "node" that advertises capabilities over WebSocket.

**macOS Node:**
- Menu bar control plane
- Voice Wake + PTT overlay
- WebChat + debug tools
- Canvas, camera, screen recording
- `system.run`, `system.notify`
- Remote gateway control

**iOS Node:**
- Canvas surface
- Voice Wake + Talk Mode
- Camera snap/clip, screen recording
- Bonjour + device pairing

**Android Node:**
- Connect tab (setup code/manual)
- Chat sessions, voice tab
- Canvas, camera/screen recording
- Android device commands (notifications, location, SMS, photos, contacts, calendar, motion, app update)

**Key Files/Locations:**
- `apps/macos/Sources/` - macOS app (Swift)
- `apps/ios/Sources/` - iOS app (Swift)
- `apps/android/app/src/` - Android app (Kotlin)
- `src/node-host/` - Node host implementation
- `docs/platforms/` - Platform documentation

**Priority:** SECONDARY

---

### 9. Skills Platform
**Description:** Extensible skill system that allows adding capabilities through packages. Supports bundled skills (core), managed skills (ClawHub), and workspace skills (user-defined).

**Key Features:**
- Bundled skills for baseline UX
- ClawHub skill registry (clawhub.ai)
- Workspace skills in `~/.openclaw/workspace/skills/`
- Skill install gating + UI
- Skills platform with install/uninstall commands

**Bundled Skills (selected):**
- `skills/acpx` - ACP transport
- `skills/anthropic/`, `skills/openai/` - Model providers
- `skills/discord/`, `skills/slack/` - Channel integrations
- `skills/spotify-player/`, `skills/sonoscli/` - Media control
- `skills/notion/`, `skills/obsidian/` - Note-taking
- `skills/github/`, `skills/gh-issues/` - GitHub integration
- `skills/summary/`, `skills/blogwatcher/` - Content skills

**Key Files/Locations:**
- `skills/` - Skill packages (53 directories)
- `src/commands/skills-cli.*` - Skills CLI
- `docs/tools/skills` - Skills documentation
- `docs/tools/plugin` - Plugin documentation

**Priority:** SECONDARY

---

### 10. Security & Sandboxing
**Description:** Comprehensive security model with secure defaults, pairing-based access control, per-session sandboxing (Docker), and fine-grained tool permissions.

**Key Features:**
- DM pairing policy (pairing/open modes)
- Allowlist-based access control (`allowFrom`)
- Per-session Docker sandboxing for non-main sessions
- Tool allowlist/denylist per session type
- Elevated bash access control (`/elevated on|off`)
- Secrets management (`openclaw secrets`)
- Security documentation and doctor checks

**Key Files/Locations:**
- `src/security/` - Security implementation (40 files)
- `src/secrets/` - Secrets management
- `src/agents/allowlists/` - Allowlist implementations
- `docs/gateway/security` - Security documentation

**Priority:** SECONDARY

---

### 11. Media Pipeline
**Description:** Unified media processing pipeline for images, audio, and video with transcription hooks, size caps, and temp file lifecycle management.

**Key Features:**
- Image processing and thumbnail generation
- Audio/video transcription hooks
- Media size caps
- Temp file lifecycle
- Screen recording capture
- Camera snap/clip

**Key Files/Locations:**
- `src/media/` - Media pipeline (50 files)
- `src/media-understanding/` - Media understanding (58 files)
- `src/image-generation/` - Image generation
- `extensions/openai-whisper/`, `extensions/openai-whisper-api/` - Whisper transcription
- `docs/nodes/images`, `docs/nodes/audio` - Media documentation

**Priority:** SECONDARY

---

### 12. Automation (Cron, Webhooks, MCP)
**Description:** Automation capabilities including scheduled cron jobs, webhook triggers, and MCP (Model Context Protocol) integration via mcporter bridge.

**Key Features:**
- Cron jobs with scheduled wakeups
- Webhook endpoints and triggers
- Gmail Pub/Sub integration
- MCP server integration via mcporter
- Session automation tools

**Key Files/Locations:**
- `src/cron/` - Cron implementation (79 files)
- `src/cli/cron-cli/` - Cron CLI
- `src/webhooks-cli.ts` - Webhook CLI
- `extensions/mcporter/` - MCP bridge
- `docs/automation/cron-jobs`, `docs/automation/webhook` - Automation docs

**Priority:** SECONDARY

---

## Summary Table

| Feature | Description | Priority | Key Locations |
|---------|-------------|----------|---------------|
| Multi-Channel Messaging | 20+ messaging platforms | CORE | `src/channels/`, `extensions/*/` |
| Gateway Control Plane | Local WebSocket control hub | CORE | `src/gateway/`, `src/infra/` |
| AI Agent Runtime | Pi agent with tool/stream support | CORE | `src/agents/` |
| Voice Wake + Talk | Voice activation on mobile/desktop | CORE | `src/tts/`, `extensions/elevenlabs/` |
| Live Canvas | Visual agent workspace | CORE | `src/canvas-host/`, `apps/*/Sources/Canvas/` |
| Browser Control | CDP-controlled browser | CORE | `src/browser/` |
| CLI Interface | Full command-line interface | CORE | `src/cli/`, `openclaw.mjs` |
| Device Nodes | macOS/iOS/Android companions | SECONDARY | `apps/*/`, `src/node-host/` |
| Skills Platform | Extensible capability packages | SECONDARY | `skills/`, `src/commands/skills-cli.*` |
| Security & Sandboxing | Secure defaults, Docker isolation | SECONDARY | `src/security/`, `src/secrets/` |
| Media Pipeline | Image/audio/video processing | SECONDARY | `src/media/`, `src/media-understanding/` |
| Automation | Cron, webhooks, MCP | SECONDARY | `src/cron/`, `extensions/mcporter/` |

---

## Cross-Reference: README Claims vs. Actual Structure

| README Claim | Verified Location |
|--------------|-------------------|
| "WhatsApp, Telegram, Slack, Discord..." | `extensions/whatsapp/`, `extensions/telegram/`, `extensions/slack/`, `extensions/discord/` |
| "Voice Wake + Talk Mode" | `src/tts/`, `extensions/elevenlabs/`, `extensions/voice-call/` |
| "Live Canvas + A2UI" | `src/canvas-host/a2ui/` |
| "Browser control" | `src/browser/` (139 files) |
| "Pi agent runtime" | `src/agents/acp-spawn.ts`, `src/agents/agent-command.ts` |
| "macOS menu bar app" | `apps/macos/Sources/OpenClawApp.swift` |
| "iOS/Android nodes" | `apps/ios/Sources/`, `apps/android/app/src/` |
| "Skills platform" | `skills/` (53 skill directories) |
| "MCP support via mcporter" | `extensions/mcporter/` |
| "Security defaults" | `src/security/` (40 files), `src/secrets/` (58 files) |

---

## Architecture Overview

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
