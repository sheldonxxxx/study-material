# Nanoclaw Project Topology

**Project:** Nanoclaw - Personal Claude assistant
**Type:** Node.js/TypeScript daemon with skill-based channel system
**Repo:** `/Users/sheldon/Documents/claw/reference/nanoclaw`

## Project Type

A **daemon/service** application that acts as a personal Claude assistant. It runs as a background service (macOS launchd or Linux systemd), listens to messages from multiple channels (WhatsApp, Telegram, Slack, Discord, Gmail), and routes them to Claude Agent SDK running inside containers (Linux VMs).

## Directory Structure

```
nanoclaw/
├── src/                    # Main application source
│   ├── index.ts           # MAIN ENTRY POINT - orchestrator, state, message loop
│   ├── channels/          # Channel system (self-registering at startup)
│   │   ├── registry.ts    # Channel factory registry
│   │   └── index.ts       # Channel exports
│   ├── config.ts          # Configuration (triggers, paths, intervals)
│   ├── db.ts              # SQLite operations
│   ├── ipc.ts             # IPC watcher and task processing
│   ├── router.ts          # Message formatting and outbound routing
│   ├── container-runner.ts # Spawns agent containers with mounts
│   ├── container-runtime.ts # Container lifecycle management
│   ├── task-scheduler.ts  # Scheduled task execution
│   ├── sender-allowlist.ts # Access control
│   ├── group-queue.ts     # Group message queuing
│   ├── group-folder.ts    # Per-group folder resolution
│   ├── remote-control.ts  # Remote control functionality
│   ├── types.ts           # TypeScript type definitions
│   ├── logger.ts          # Pino logger setup
│   ├── env.ts             # Environment variables
│   ├── timezone.ts        # Timezone utilities
│   └── *.test.ts          # Test files co-located with source
│
├── setup/                  # Setup/installation scripts
│   ├── index.ts           # Setup entry point
│   ├── register.ts        # Service registration
│   ├── service.ts         # Service management
│   ├── platform.ts        # Platform detection
│   ├── environment.ts     # Environment setup
│   ├── mounts.ts          # Filesystem mount configuration
│   ├── container.ts       # Container setup
│   └── verify.ts          # Installation verification
│
├── container/              # Agent container build
│   ├── Dockerfile         # Container image definition
│   ├── build.sh          # Container build script
│   ├── agent-runner/      # Code run inside containers
│   │   └── src/          # Agent runner source
│   └── skills/           # Container skills (loaded at runtime)
│       ├── agent-browser/
│       ├── capabilities/
│       ├── slack-formatting/
│       └── status/
│
├── .claude/               # Claude skills (operational/utility skills)
│   ├── skills/           # Feature, utility, operational skills
│   │   ├── setup/        # First-time installation
│   │   ├── customize/    # Adding channels
│   │   ├── debug/       # Troubleshooting
│   │   ├── claw/        # Utility skill
│   │   └── add-*/       # Feature skills (add-telegram, add-whatsapp, etc.)
│   └── src/
│       ├── index.ts     # Skills entry point
│       └── ipc-mcp-stdio.ts
│
├── groups/               # Group-based isolation (per-group memory)
│   ├── global/          # Global group
│   └── main/            # Main group
│
├── docs/                # Documentation
│   ├── SPEC.md          # Detailed specification
│   ├── REQUIREMENTS.md  # Architecture decisions
│   ├── SECURITY.md      # Security documentation
│   ├── SDK_DEEP_DIVE.md # SDK analysis
│   └── docker-sandboxes.md
│
├── config-examples/     # Example configurations
├── launchd/             # macOS launchd plist
├── scripts/            # Utility scripts
├── assets/             # Images (logos, icons)
├── repo-tokens/        # Repository tokens
├── package.json        # Main npm config
├── tsconfig.json       # TypeScript config
├── vitest.config.ts    # Test config
└── README.md           # Project documentation
```

## Entry Points

| Entry Point | Purpose |
|------------|---------|
| `src/index.ts` | **Primary entry point** - Orchestrates state, message loop, agent invocation |
| `npm run dev` | Development mode (`tsx src/index.ts`) |
| `npm start` | Production mode (`node dist/index.js`) |
| `npm run build` | Compile TypeScript to `dist/` |
| `setup/index.ts` | Initial setup and installation |
| `container/build.sh` | Build agent container image |

## Key Source Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/index.ts` | ~600 | Main orchestrator: state management, message loop, agent invocation |
| `src/db.ts` | ~500 | SQLite database operations |
| `src/container-runner.ts` | ~600 | Spawns/manages agent containers |
| `src/ipc.ts` | ~400 | IPC watching and task processing |
| `src/router.ts` | ~40 | Message formatting and outbound routing |
| `src/channels/registry.ts` | ~20 | Channel factory registry (self-registration) |
| `src/task-scheduler.ts` | ~250 | Cron-based scheduled task execution |

## Directory Layout Patterns

- **Source co-location**: Test files sit next to source (`*.test.ts` alongside `*.ts`)
- **Skills system**: Skills are directories under `.claude/skills/` with `index.ts` and/or `SKILL.md`
- **Container skills**: Separate `container/skills/` for runtime-loaded skills
- **Channel registry**: Channels self-register via `src/channels/index.js` import in `src/index.ts`

## Configuration

- **Trigger patterns**: Configured in `src/config.ts` (DEFAULT_TRIGGER, getTriggerPattern)
- **Database**: SQLite via `better-sqlite3`, database file path in config
- **Logging**: Pino logger, configured in `src/logger.ts`
- **Credentials**: Managed via OneCLI gateway (no direct secret injection)

## Build Output

- TypeScript compiles to `dist/` directory
- Main entry: `dist/index.js`
- Start command: `node dist/index.js`
