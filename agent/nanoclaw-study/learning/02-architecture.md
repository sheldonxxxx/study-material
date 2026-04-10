# NanoClaw Architecture

**Project:** NanoClaw - Personal Claude assistant
**Analysis Date:** 2026-03-26
**Source:** `/Users/sheldon/Documents/claw/reference/nanoclaw`

## Architectural Pattern

NanoClaw uses a **Layered + Event-Driven Architecture** combining:

1. **Layered separation**: Channels (input) -> Core Orchestrator -> Container Execution (sandbox) -> Channels (output)
2. **Event-driven polling**: Message loop polls channels periodically, IPC watcher polls filesystem
3. **Registry/Plugin pattern**: Channel factories self-register at startup
4. **Actor-like concurrency**: GroupQueue manages per-group container lifecycle with bounded concurrency

## High-Level Component Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         HOST PROCESS                                 │
│                                                                      │
│  ┌──────────────┐    ┌─────────────────┐    ┌──────────────────┐  │
│  │   Channels   │───▶│   Orchestrator  │───▶│  GroupQueue      │  │
│  │  (WhatsApp,  │    │   (index.ts)    │    │  (concurrency    │  │
│  │   Telegram,  │◀───│                 │◀───│   management)     │  │
│  │   Slack...)  │    │                 │    │                  │  │
│  └──────────────┘    └────────┬────────┘    └────────┬─────────┘  │
│                               │                      │             │
│                    ┌──────────▼──────────┐   ┌───────▼────────┐   │
│                    │   Database (SQLite) │   │ ContainerRunner│   │
│                    │   - messages        │   │                │   │
│                    │   - sessions       │   │  - Docker spawn │   │
│                    │   - tasks          │   │  - Volume mounts│   │
│                    │   - registered_    │   │  - IPC stdin/  │   │
│                    │     groups         │   │    stdout       │   │
│                    └────────────────────┘   └────────┬────────┘   │
│                                                      │             │
│  ┌──────────────┐    ┌─────────────────┐            │             │
│  │   IPC Watcher│    │  TaskScheduler  │            │             │
│  │  (file-based │    │  (cron/interval)│            │             │
│  │   commands)  │    │                 │            │             │
│  └──────────────┘    └─────────────────┘            │             │
└──────────────────────────────────────────────────────┼─────────────┘
                                                       │
                               ┌───────────────────────┘
                               ▼
                    ┌──────────────────────┐
                    │   CONTAINER (Docker) │
                    │                      │
                    │  ┌────────────────┐  │
                    │  │  Agent Runner  │  │
                    │  │  (index.ts)    │  │
                    │  │                │  │
                    │  │  - Claude SDK  │  │
                    │  │  - IPC polling │  │
                    │  │  - MCP server  │  │
                    │  └────────────────┘  │
                    │                      │
                    │  Mounts:            │
                    │  - /workspace/group │
                    │  - /workspace/ipc   │
                    │  - /home/node/      │
                    │    .claude          │
                    │  - /workspace/project│
                    └──────────────────────┘
```

## Architectural Decisions

### AD-01: Channel Self-Registration via Side-Effect Imports

Channels are loaded via side-effect imports in `src/channels/index.js`. Each channel module calls `registerChannel()` at import time, eliminating manual registration boilerplate.

```typescript
// src/channels/index.ts (empty file - imports trigger registration)
// Individual channels register themselves when imported:
// import './whatsapp/index.js';  // registers itself
// import './telegram/index.js';  // registers itself
```

**Decision:** This pattern keeps channel addition purely additive - add a new directory, import it, done. No central registry update required.

**Reference:** `src/channels/registry.ts`

### AD-02: Per-Group Isolation via Filesystem Namespace

Each group (chat conversation) has isolated:
- `groups/{folder}/` - group-specific working directory
- `data/sessions/{folder}/.claude/` - isolated Claude session state
- `data/ipc/{folder}/` - isolated IPC namespace (input/, tasks/, messages/)

**Decision:** Filesystem isolation is the primary security boundary. Agents cannot access other groups' data even if compromised.

**Reference:** `src/container-runner.ts:119-178` (mount construction)

### AD-03: Container as Execution Sandbox

Agent code executes inside Docker containers, not in the host process. The host spawns containers on demand, mounts group-specific namespaces, and communicates via stdin/stdout JSON.

**Decision:** Containers provide hard isolation with resource limits, independent process state, and clean teardown via `--rm` flag.

**Reference:** `src/container-runner.ts:277-671` (runContainerAgent)

### AD-04: IPC via Filesystem (Not Sockets or Pipes)

Container-to-host communication uses JSON files written to the group's IPC directory. The host polls this directory at `IPC_POLL_INTERVAL` (1000ms).

**Decision:** Filesystem IPC is simple, debuggable, survives container restarts, and naturally supports per-group namespacing.

**Reference:** `src/ipc.ts:30-155` (startIpcWatcher)

### AD-05: Message-Driven Container Lifecycle

Containers are not persistent daemons. They spin up when messages arrive, process them, then enter idle-wait mode (up to `IDLE_TIMEOUT` = 30min) for follow-up messages before graceful shutdown.

**Decision:** This balances startup latency (cold start ~2-5s) against resource usage. Follow-up messages within the idle window reuse the same container.

**Reference:** `src/group-queue.ts:148-154` (notifyIdle), `src/index.ts` (message polling loop)

### AD-06: Credentials via OneCLI Gateway

API keys, OAuth tokens, and secrets are NOT passed via environment variables or files. Instead, the OneCLI gateway intercepts HTTPS traffic and injects credentials at request time.

**Decision:** This prevents credential leakage through `docker inspect`, log files, or container compromise.

**Reference:** `src/container-runner.ts:236-249` (onecli.applyContainerConfig)

## Module Responsibilities

| Module | Responsibility | Key Types |
|--------|----------------|-----------|
| `src/index.ts` | Orchestrator: message loop, state management, channel setup, container invocation | Main entry point |
| `src/db.ts` | SQLite persistence for messages, tasks, sessions, groups | CRUD for all entities |
| `src/channels/registry.ts` | Channel factory registry (self-registration) | `ChannelFactory` |
| `src/container-runner.ts` | Docker container lifecycle, volume mounts, IPC stdin/stdout | `VolumeMount`, `ContainerInput`, `ContainerOutput` |
| `src/container-runtime.ts` | Container runtime abstraction (Docker CLI) | Runtime-agnostic interface |
| `src/group-queue.ts` | Per-group container concurrency, message/task queuing | `GroupState`, `QueuedTask` |
| `src/ipc.ts` | IPC command processing (task CRUD, group registration) | `IpcDeps` |
| `src/task-scheduler.ts` | Cron/interval/once task execution | `SchedulerDependencies`, `computeNextRun()` |
| `src/router.ts` | Message formatting (XML), channel routing | `formatMessages()`, `findChannel()`, `routeOutbound()` |
| `src/config.ts` | Configuration values, trigger patterns, paths | Constants and `getTriggerPattern()` |
| `src/types.ts` | TypeScript type definitions | `Channel`, `RegisteredGroup`, `NewMessage`, `ScheduledTask` |

## Communication Patterns

### 1. Channel -> Orchestrator (Inbound)

```
Channel receives message from external platform
  -> onMessage(chatJid, NewMessage) callback
  -> storeMessage() -> SQLite
  -> queue.enqueueMessageCheck(chatJid)
```

**Reference:** `src/index.ts` (channel setup with onMessage callback)

### 2. Orchestrator -> Container (via GroupQueue)

```
Message polling loop (index.ts)
  -> getNewMessages() from SQLite
  -> formatMessages() -> XML string
  -> queue.sendMessage() OR queue.enqueueMessageCheck()

GroupQueue.runForGroup()
  -> processGroupMessages()
  -> runAgent()
  -> runContainerAgent()
    -> spawn('docker run -i --rm ...')
    -> stdin.write(JSON.stringify(ContainerInput))
    -> stdout streaming via OUTPUT_MARKER pairs

Container stdin/stdout:
  - Input: ContainerInput JSON
  - Output: ---NANOCLAW_OUTPUT_START---\n{JSON}\n---NANOCLAW_OUTPUT_END---
```

**Reference:** `src/container-runner.ts:33-35` (OUTPUT_START_MARKER, OUTPUT_END_MARKER)

### 3. Orchestrator <-> Container (Follow-up Messages via IPC Files)

```
Host writes IPC file:
  data/ipc/{groupFolder}/input/{timestamp}-{random}.json
  -> { type: 'message', text: '...' }

Container polls IPC directory:
  drainIpcInput() -> returns accumulated message texts
  -> piped into MessageStream (AsyncIterable)

Host signals close:
  data/ipc/{groupFolder}/input/_close (sentinel file)
  Container detects -> exits query loop
```

**Reference:** `src/group-queue.ts:160-194` (sendMessage, closeStdin)

### 4. IPC Commands (Container -> Host)

```
Container writes to IPC:
  data/ipc/{sourceGroup}/tasks/{timestamp}.json
  -> { type: 'schedule_task', prompt, schedule_type, schedule_value, targetJid }
  -> OR { type: 'register_group', jid, name, folder, trigger }

Host IPC watcher:
  processTaskIpc() -> handles schedule_task, pause_task, resume_task, cancel_task, update_task, refresh_groups, register_group
```

**Reference:** `src/ipc.ts:157-464` (processTaskIpc switch statement)

### 5. Scheduler -> Container

```
TaskScheduler (task-scheduler.ts)
  -> getDueTasks() from SQLite
  -> queue.enqueueTask()
  -> GroupQueue.runTask()
  -> runContainerAgent(isScheduledTask: true)
```

**Reference:** `src/task-scheduler.ts:78+`

## Security Model

### Isolation Layers

1. **Filesystem Isolation**: Each group has isolated `groups/{folder}/`, `data/sessions/{folder}/`, `data/ipc/{folder}/`
2. **Credential Isolation**: OneCLI gateway injects credentials via environment; containers never see `.env`
3. **IPC Isolation**: Per-group IPC directories prevent cross-group privilege escalation
4. **Mount Allowlist**: Additional mounts validated against `~/.config/nanoclaw/mount-allowlist.json`

### Host Protection

- Main group mounts project root **read-only** (`src/`, `dist/`, `package.json` not writable)
- `.env` shadowed with `/dev/null` in containers
- `CONTAINER_RUNTIME_BIN` is the only runtime (Docker), hardcoded abstraction
- Agent runner source copied to per-group writable location (agents can customize without affecting other groups)

### IPC Authorization

IPC commands carry `sourceGroup` identity derived from the directory path (not from the JSON content):

```typescript
// src/ipc.ts:128-129
// Pass source group identity to processTaskIpc for authorization
await processTaskIpc(data, sourceGroup, isMain, deps);
```

Authorization rules:
- Non-main groups can only schedule tasks for themselves
- Only main group can register new groups
- Only main group can refresh group metadata

## Database Schema

```sql
chats              -- Chat metadata (jid, name, last_message_time, channel, is_group)
messages           -- Full message content (id, chat_jid, sender, content, timestamp, is_from_me, is_bot_message)
scheduled_tasks    -- Task definitions (id, group_folder, chat_jid, prompt, schedule_type, schedule_value, next_run, status, context_mode, script)
task_run_logs      -- Task execution history (task_id, run_at, duration_ms, status, result, error)
router_state       -- Key-value state (last_timestamp, last_agent_timestamp)
sessions           -- Group -> session_id mapping (group_folder, session_id)
registered_groups  -- Group configuration (jid, name, folder, trigger_pattern, is_main, requires_trigger, container_config)
```

**Reference:** `src/db.ts:17-85` (createSchema)

## Key Interfaces

### Channel Interface

```typescript
// src/types.ts:83-94
interface Channel {
  name: string;
  connect(): Promise<void>;
  sendMessage(jid: string, text: string): Promise<void>;
  isConnected(): boolean;
  ownsJid(jid: string): boolean;
  disconnect(): Promise<void>;
  setTyping?(jid: string, isTyping: boolean): Promise<void>;  // optional
  syncGroups?(force: boolean): Promise<void>;                 // optional
}
```

### RegisteredGroup Interface

```typescript
// src/types.ts:35-43
interface RegisteredGroup {
  name: string;           // Display name
  folder: string;         // Isolated filesystem directory
  trigger: string;        // Regex pattern to activate non-main groups
  added_at: string;       // Registration timestamp
  containerConfig?: ContainerConfig;
  requiresTrigger?: boolean;  // Default true for groups
  isMain?: boolean;          // True = main control group
}
```

### ContainerInput / ContainerOutput

```typescript
// src/container-runner.ts:37-53
interface ContainerInput {
  prompt: string;
  sessionId?: string;
  groupFolder: string;
  chatJid: string;
  isMain: boolean;
  isScheduledTask?: boolean;
  assistantName?: string;
  script?: string;
}

interface ContainerOutput {
  status: 'success' | 'error';
  result: string | null;
  newSessionId?: string;
  error?: string;
}
```

## Directory Structure

```
nanoclaw/
├── src/                    # Main application source
│   ├── index.ts           # MAIN ENTRY POINT - orchestrator, state, message loop
│   ├── channels/          # Channel system (self-registering at startup)
│   │   ├── registry.ts   # Channel factory registry
│   │   └── index.ts      # Channel exports
│   ├── config.ts         # Configuration (triggers, paths, intervals)
│   ├── db.ts             # SQLite operations
│   ├── ipc.ts            # IPC watcher and task processing
│   ├── router.ts         # Message formatting and outbound routing
│   ├── container-runner.ts # Spawns agent containers with mounts
│   ├── container-runtime.ts # Container lifecycle management
│   ├── task-scheduler.ts # Scheduled task execution
│   ├── group-queue.ts    # Per-group container concurrency
│   ├── group-folder.ts   # Per-group folder resolution
│   ├── sender-allowlist.ts # Access control
│   ├── remote-control.ts # Remote control functionality
│   ├── types.ts          # TypeScript type definitions
│   ├── logger.ts         # Pino logger setup
│   ├── env.ts            # Environment variables
│   └── timezone.ts       # Timezone utilities
├── setup/                 # Setup/installation scripts
├── container/             # Agent container build
│   ├── Dockerfile        # Container image definition
│   ├── build.sh         # Container build script
│   ├── agent-runner/    # Code run inside containers
│   └── skills/          # Container skills (loaded at runtime)
├── .claude/              # Claude skills (operational/utility skills)
├── groups/               # Group-based isolation (per-group memory)
├── data/                 # Runtime data (sessions, IPC, logs)
├── docs/                 # Documentation
├── config-examples/      # Example configurations
├── launchd/              # macOS launchd plist
└── scripts/              # Utility scripts
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
