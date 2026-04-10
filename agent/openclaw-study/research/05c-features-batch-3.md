# OpenClaw Feature Batch 3 Report: CLI, Device Nodes, Skills Platform

**Date:** 2026-03-26
**Repo:** `/Users/sheldon/Documents/claw/reference/openclaw`
**Version:** 2026.3.24

---

## Feature 7: Command-Line Interface (CLI)

### Overview
The OpenClaw CLI is a full-featured command-line interface (`openclaw`) built on Node.js 22+ with sophisticated bootstrap logic, module compile caching, and graceful fallback mechanisms.

### Entry Point Architecture

**Primary Entry: `openclaw.mjs`** (181 lines)
- Shebang: `#!/usr/bin/env node`
- Node.js version guard: Requires 22.12.0+
- Module compile cache enabled via `module.enableCompileCache()`
- Fast-path for `--help` with precomputed root help text (loads from `dist/cli-startup-metadata.json`)
- Graceful fallback chain: `dist/entry.js` -> `dist/entry.mjs` -> `src/entry.ts` (source)
- Error messages guide users to rebuild or use npm global install

### CLI Bootstrap Flow
```
openclaw.mjs
  -> installProcessWarningFilter()
  -> tryImport(dist/entry.js)
  -> tryImport(dist/entry.mjs)
  -> throw Error (missing dist guidance)
```

### Key CLI Commands (from topology)
| Command | Location | Purpose |
|---------|----------|---------|
| `openclaw onboard` | Setup wizard | Step-by-step onboarding |
| `openclaw gateway` | gateway-cli/ | Gateway management |
| `openclaw agent` | Agent commands | Agent interaction |
| `openclaw message send` | Messaging | Direct messaging |
| `openclaw channels` | channels-cli | Channel management |
| `openclaw devices` | nodes-cli | Device/node pairing |
| `openclaw plugins` | plugins-cli | Plugin lifecycle |
| `openclaw config` | config-cli | Configuration |
| `openclaw doctor` | Diagnostics | Troubleshooting |

### CLI Implementation Structure (196 files in `src/cli/`)
- `src/cli/run-main.ts` - Main CLI runner
- `src/cli/program/` - Commander.js root program
- `src/cli/gateway-cli/` - Gateway subcommands
- `src/cli/plugins-cli/` - Plugin management
- `src/cli/config-cli.ts` - Config management
- `src/cli/secrets-cli.ts` - Secrets management
- `src/cli/nodes-cli/` - Node management
- `src/cli/pairing-cli.ts` - Device pairing

### Notable Patterns
1. **Precomputed Help Text**: `--help` fast-path loads from JSON instead of running full CLI bootstrap
2. **Warning Filter**: Bootstrap warnings are filtered to match TypeScript runtime behavior
3. **Module Resolution**: Uses ESM-only imports with `await import()` for lazy loading
4. **Source Fallback**: If dist is missing but src exists, provides actionable error message

### Technical Debt / Observations
- The CLI relies heavily on Commander.js patterns
- No visible CLI test files in the initial directory listing (though source has `*.test.ts` colocation)
- Complex version detection and fallback logic in entry point

---

## Feature 8: Device Nodes (macOS/iOS/Android Companion Apps)

### Overview
Companion apps for macOS, iOS, and Android that pair with the Gateway to expose device-local capabilities. Each device runs as a "node" advertising capabilities over WebSocket.

### macOS App (`apps/macos/Sources/OpenClaw/`)

**Architecture: Menu Bar Extra App (SwiftUI)**
- Uses `MenuBarExtra` for menu bar presence
- `NSApplicationDelegate` with `@MainActor` SwiftUI app
- Extensive modules: VoiceWake, Canvas, Gateway, Sessions, Settings

**Key Components:**
| Component | Files | Purpose |
|-----------|-------|---------|
| VoiceWake | 15+ files | Wake word detection, PTT overlay |
| Canvas | 12+ files | Canvas window, scheme handler, file watcher |
| Gateway | 30+ files | Launch agent, connection, discovery |
| Sessions | 15+ files | Session management, menu preview |
| Settings | 40+ files | Config UI, schema support |

**Menu Bar Structure:**
- `MenuBar.swift` - Main menu bar app
- `MenuContentView.swift` - Menu content
- `MenuSessionsInjector.swift` - Session injection
- `MenuHighlightedHostView.swift` - Highlighted items
- `HoverHUD.swift` - Hover HUD overlay

**Gateway Management:**
- `GatewayProcessManager.swift` - Process lifecycle
- `GatewayDiscoveryMenu.swift` - Bonjour discovery
- `GatewayLaunchAgentManager.swift` - launchd integration
- `RemoteTunnelManager.swift` - Tailscale integration

### iOS App (`apps/ios/Sources/`)

**Architecture: SwiftUI with UIKit bridges**
- Uses `Observation` framework (`@Observable`) per CLAUDE.md guidelines
- `OpenClawAppDelegate` for push notifications

**Key Components:**
| Component | Purpose |
|-----------|---------|
| `RootCanvas.swift` | Main canvas surface |
| `Gateway/` | Gateway connection management |
| `Voice/` | Voice Wake + Talk Mode |
| `Camera/` | Camera capture |
| `Chat/` | Chat sessions |
| `Onboarding/` | Setup wizard |

**iOS-Specific Features:**
- `VoiceWakeManager` - Voice wake detection
- `TalkModeController` - Voice conversation
- `RootTabs.swift` - Tab navigation
- APNs push notification handling via `OpenClawAppDelegate`

### Android App (`apps/android/app/src/main/java/ai/openclaw/app/`)

**Architecture: Kotlin with Jetpack Compose**

**Main Components:**
| Component | Purpose |
|-----------|---------|
| `NodeApp.kt` | Application class with runtime singleton |
| `MainActivity.kt` | Main entry with Compose UI |
| `MainViewModel.kt` | View model (41KB) |
| `NodeRuntime.kt` | Core runtime (41KB) |

**Node Handlers (`node/` directory - 25 files):**
- `InvokeCommandRegistry.kt` - Command/capability advertisement
- `InvokeDispatcher.kt` - Command routing
- `ConnectionManager.kt` - Gateway TLS/connection handling
- `GatewayEventHandler.kt` - Wake words sync, events
- `CameraHandler.kt`, `LocationHandler.kt`, `SmsHandler.kt`, etc.

**Android-Specific Capabilities:**
```
InvokeCommandRegistry.advertisedCapabilities:
- Canvas (always)
- Device (always)
- Notifications (always)
- System (always)
- Camera (CameraEnabled)
- SMS (SmsAvailable)
- VoiceWake (VoiceWakeEnabled)
- Location (LocationEnabled)
- Photos (always)
- Contacts (always)
- Calendar (always)
- Motion (MotionAvailable)
- CallLog (CallLogAvailable)
```

### Shared Code: OpenClawKit (`apps/shared/OpenClawKit/`)

**Swift Package for cross-platform sharing:**
```
Sources/
├── OpenClawKit/ (76 files)
│   ├── GatewayDiscoveryBrowserSupport.swift
│   ├── GatewayNodeSession.swift
│   ├── DeviceAuthStore.swift
│   ├── CanvasA2UICommands.swift
│   └── ... (72 more)
├── OpenClawChatUI/ (16 files)
└── OpenClawProtocol/ (5 files)
```

**Shared Protocol:**
- `GatewayChannel.swift` - WebSocket communication
- `DeviceAuthPayload.swift` - Auth handshake
- `GatewayTLSPinning.swift` - TLS certificate pinning
- `NodeError.swift` - Error types

### Device Pairing Flow
1. App discovers gateway via Bonjour (`GatewayDiscovery`)
2. QR code / setup code generated via `openclaw qr`
3. TLS handshake with certificate pinning
4. Device auth store saves credentials
5. `InvokeCommandRegistry` advertises runtime capabilities
6. Gateway approves device via `openclaw devices approve`

### Notable Solutions
1. **TLS Pinning**: Supports both automatic TOFU and manual fingerprint verification
2. **Capability Advertisement**: Runtime flags determine which commands are available
3. **Invoke Dispatcher**: Central routing for all node commands
4. **Foreground Service**: Android uses `NodeForegroundService` for background operation
5. **SecurePrefs**: Android encrypted preferences for sensitive data

---

## Feature 9: Skills Platform (Extensible Capability Packages)

### Overview
Skills are modular, self-contained packages that extend the AI agent's capabilities through specialized knowledge, workflows, and tool integrations. They follow a markdown-first design philosophy.

### Skill Anatomy

**Directory Structure:**
```
skills/<name>/
├── SKILL.md (required) - YAML frontmatter + markdown instructions
├── scripts/ (optional) - Executable code (Python/Bash/etc.)
├── references/ (optional) - Documentation for context loading
├── assets/ (optional) - Files for output (templates, icons)
└── bin/ (optional) - Binary executables
```

**SKILL.md Frontmatter Schema:**
```yaml
---
name: skill-name
description: "When to use this skill + specific triggers"
metadata:
  openclaw:
    emoji: "..."          # Optional icon
    requires:             # Dependencies
      bins: ["gh"]        # CLI tools needed
      env: ["API_KEY"]    # Environment variables
      config: ["path"]     # Config files
    install:              # Auto-install instructions
      - id: brew
        kind: brew
        formula: gh
---
```

**Progressive Disclosure Design:**
1. **Metadata** (~100 words) - Always in context for triggering
2. **SKILL.md body** (<5k words) - Loaded after skill triggers
3. **Bundled resources** - Loaded as needed by agent

### Skill Categories (53 total)

| Category | Examples |
|----------|----------|
| **Coding** | `coding-agent`, `github`, `gh-issues` |
| **Communication** | `slack`, `discord`, `imsg`, `wacli` |
| **Media** | `spotify-player`, `sonoscli`, `songsee`, `video-frames` |
| **Notes** | `notion`, `obsidian`, `apple-notes`, `bear-notes` |
| **System** | `tmux`, `himalaya`, `model-usage`, `session-logs` |
| **IoT** | `openhue`, `bluebubbles`, `blucli` |
| **Content** | `summarize`, `blogwatcher`, `xurl`, `nano-pdf` |
| **Platform** | `clawhub`, `skill-creator`, `mcporter`, `node-connect` |

### Skill Creation Process (from `skill-creator`)

1. **Understand with examples** - Concrete usage patterns
2. **Plan contents** - Scripts, references, assets needed
3. **Initialize** - `scripts/init_skill.py <name> --path skills/`
4. **Edit** - Write SKILL.md and implement resources
5. **Package** - `scripts/package_skill.py <path>` (validates + creates `.skill` file)
6. **Iterate** - Based on real usage

### ClawHub Integration

**ClawHub CLI** (`clawhub`):
```bash
clawhub search "postgres backups"    # Search registry
clawhub install my-skill            # Install from registry
clawhub update my-skill             # Hash-based update
clawhub publish ./my-skill          # Publish to registry
```

**Skill Registry:** https://clawhub.ai

### Example Skills

**GitHub (`skills/github/`):**
- Uses `gh` CLI for GitHub operations
- Commands: PR list/check/view/create/merge, issues, CI runs
- Setup: `gh auth login`
- JSON output with `--json --jq` filtering

**Notion (`skills/notion/`):**
- API-based page/database operations
- Requires `NOTION_API_KEY` in `~/.config/notion/api_key`
- Uses API version `2025-09-03`
- Note: "databases" renamed to "data sources" in newer API

**Slack (`skills/slack/`):**
- React, pin/unpin, send/edit/delete messages
- JSON tool action format
- Action groups: reactions, messages, pins, memberInfo, emojiList

**Coding Agent (`skills/coding-agent/`):**
- Delegates to Codex, Claude Code, or Pi
- PTY mode required for Codex/Pi/OpenCode
- Claude Code uses `--print --permission-mode bypassPermissions`
- Background execution with `process` tool actions

### Notable Patterns

1. **Bash-First**: Many skills are bash command wrappers (e.g., `github`, `weather`, `summarize`)
2. **CLI Integration**: Skills often wrap existing CLI tools (`gh`, `mcporter`, `clawhub`)
3. **No Code in Skills**: Most skills are pure documentation with bash examples
4. **Token Efficiency**: SKILL.md kept lean with references to external docs
5. **Metadata-Driven**: Frontmatter drives skill selection and dependency checking

### Technical Observations

1. **Skills are Documentation-Heavy**: 53 skills but most contain only `SKILL.md`
2. **Bundled Resources Rare**: Only a few skills have `scripts/`, `references/`, or `bin/`
3. **Skill Creator as Meta-Skill**: `skill-creator` provides comprehensive authoring guidance
4. **Workspace Skills**: Users can add skills to `~/.openclaw/workspace/skills/`
5. **No Runtime Skill Code**: Skills are instructions for the AI, not executable code (except bundled scripts)

---

## Cross-Feature Integration

### CLI -> Device Nodes
- `openclaw qr` generates pairing QR codes
- `openclaw devices list` shows pending pairings
- `openclaw devices approve --latest` approves devices

### Device Nodes -> Gateway
- WebSocket connection to `ws://127.0.0.1:18789` (default)
- TLS pinning with fingerprint verification
- Capability advertisement via `InvokeCommandRegistry`
- Wake word sync via `GatewayEventHandler`

### Skills -> Agent
- Skills provide procedural knowledge for AI agent
- Triggered by metadata description matching user requests
- Bundled scripts executed without context loading
- ClawHub enables dynamic skill installation

---

## Summary Table

| Feature | Complexity | Files/LOC | Key Technology | Standout Pattern |
|---------|------------|-----------|----------------|------------------|
| CLI | Medium | ~196 source files | Commander.js, Node 22 | Precomputed help, graceful fallback |
| Device Nodes | High | macOS: 230+ files, iOS: 30+, Android: 25+ handlers | Swift (macOS/iOS), Kotlin (Android) | InvokeCommandRegistry capability ads |
| Skills Platform | Low | 53 SKILL.md files | Markdown + bash | Progressive disclosure, token efficiency |

---

## Observations

1. **CLI Robustness**: The bootstrap is well-engineered with version checks, compile cache, and helpful error messages
2. **Device Node Sophistication**: Android's `InvokeCommandRegistry` shows thoughtful capability advertisement design
3. **Skills Philosophy**: Skills are instruction documents, not code packages - AI reads them to understand how to act
4. **Shared Code**: OpenClawKit shows good cross-platform code sharing for Swift platforms
5. **TLS Security**: Both CLI and device nodes show careful TLS handling with pinning and fingerprint verification
