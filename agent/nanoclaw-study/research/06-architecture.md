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

## Key Abstractions

### Channel Interface (`src/types.ts`)

```typescript
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

Channels are plugins that:
- Connect to external messaging platforms (WhatsApp, Telegram, Slack, Discord, Gmail)
- Deliver inbound messages via `onMessage` callback
- Send outbound messages via `sendMessage()`
- Identify which JIDs they own via `ownsJid()`

### RegisteredGroup (`src/types.ts`)

```typescript
interface RegisteredGroup {
  name: string;           // Display name
  folder: string;        // Isolated filesystem directory
  trigger: string;       // Regex pattern to activate non-main groups
  added_at: string;      // Registration timestamp
  containerConfig?: ContainerConfig;
  requiresTrigger?: boolean;  // Default true for groups
  isMain?: boolean;          // True = main control group
}
```

Groups are the core isolation unit. Each group has:
- Isolated filesystem (`groups/{folder}/`)
- Isolated Claude session (`data/sessions/{folder}/.claude/`)
- Isolated IPC namespace (`data/ipc/{folder}/`)
- Independent trigger pattern (except main group)

### ContainerInput / ContainerOutput (`src/container-runner.ts`)

```typescript
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

## Module Responsibilities

| Module | Responsibility | Key Types |
|--------|----------------|-----------|
| `src/index.ts` | Orchestrator: message loop, state management, channel setup | Main entry point |
| `src/db.ts` | SQLite persistence for messages, tasks, sessions, groups | CRUD for all entities |
| `src/channels/registry.ts` | Channel factory registry (self-registration) | `ChannelFactory` |
| `src/container-runner.ts` | Docker container lifecycle, volume mounts, IPC stdin/stdout | `VolumeMount`, `ContainerInput` |
| `src/container-runtime.ts` | Container runtime abstraction (Docker CLI) | Runtime-agnostic |
| `src/group-queue.ts` | Per-group container concurrency, message/task queuing | `GroupState` |
| `src/ipc.ts` | IPC command processing (task CRUD, group registration) | `IpcDeps` |
| `src/task-scheduler.ts` | Cron/interval/once task execution | `SchedulerDependencies` |
| `src/router.ts` | Message formatting, channel routing | `formatMessages()`, `findChannel()` |

## Component Communication

### 1. Channel -> Orchestrator (Inbound)

```
Channel receives message
  → onMessage(chatJid, NewMessage) callback
  → storeMessage() → SQLite
  → queue.enqueueMessageCheck(chatJid)
```

### 2. Orchestrator -> Container (via GroupQueue)

```
Message polling loop (index.ts)
  → getNewMessages() from SQLite
  → formatMessages() → XML string
  → queue.sendMessage() OR queue.enqueueMessageCheck()

GroupQueue.runForGroup()
  → processGroupMessages()
  → runAgent()
  → runContainerAgent()
    → spawn('docker run -i --rm ...')
    → stdin.write(JSON.stringify(ContainerInput))
    → stdout streaming via OUTPUT_MARKER pairs

Container stdin/stdout:
  - Input: ContainerInput JSON
  - Output: ---NANOCLAW_OUTPUT_START---\n{JSON}\n---NANOCLAW_OUTPUT_END---
```

### 3. Orchestrator <-> Container (Follow-up Messages via IPC Files)

```
Host writes IPC file:
  data/ipc/{groupFolder}/input/{timestamp}-{random}.json
  → { type: 'message', text: '...' }

Container polls IPC directory:
  drainIpcInput() → returns accumulated message texts
  → piped into MessageStream (AsyncIterable)

Host signals close:
  data/ipc/{groupFolder}/input/_close (sentinel file)
  Container detects → exits query loop
```

### 4. IPC Commands (Container -> Host)

```
Container writes to IPC:
  data/ipc/{sourceGroup}/tasks/{timestamp}.json
  → { type: 'schedule_task', prompt, schedule_type, schedule_value, targetJid }
  → OR { type: 'register_group', jid, name, folder, trigger }

Host IPC watcher:
  processTaskIpc() → handles schedule_task, pause_task, resume_task, cancel_task, update_task, refresh_groups, register_group
```

### 5. Scheduler -> Container

```
TaskScheduler (task-scheduler.ts)
  → getDueTasks() from SQLite
  → queue.enqueueTask()
  → GroupQueue.runTask()
  → runContainerAgent(isScheduledTask: true)
```

## Design Patterns in Use

### 1. Registry/Factory Pattern
Channels self-register via side-effect imports in `src/channels/index.ts`:
```typescript
// channels/index.ts
import './whatsapp/index.js';  // registers itself
import './telegram/index.js';  // registers itself
```
Each channel module calls `registerChannel(name, factory)` at import time.

### 2. Strategy Pattern
`ChannelFactory` allows different channel implementations with the same interface:
```typescript
type ChannelFactory = (opts: ChannelOpts) => Channel | null;
```

### 3. Observer Pattern
`GroupQueue` notifies the orchestrator via callbacks:
```typescript
setProcessMessagesFn(fn: (groupJid: string) => Promise<boolean>)
```

### 4. AsyncIterable / Stream Pattern
`MessageStream` class implements async iteration for streaming messages to the SDK:
```typescript
class MessageStream {
  async *[Symbol.asyncIterator](): AsyncGenerator<SDKUserMessage>
  push(text: string): void
  end(): void
}
```

### 5. Template Method
`processGroupMessages()` and `runAgent()` define the skeleton; container-runner executes the steps.

### 6. Circuit Breaker
`GroupQueue` implements retry with exponential backoff:
```typescript
const BASE_RETRY_MS = 5000;
const delayMs = BASE_RETRY_MS * Math.pow(2, state.retryCount - 1);
```

### 7. Leased Concurrency
`GroupQueue` enforces `MAX_CONCURRENT_CONTAINERS` limit:
```typescript
if (this.activeCount >= MAX_CONCURRENT_CONTAINERS) {
  state.pendingMessages = true;
  this.waitingGroups.push(groupJid);
}
```

### 8. Snapshot / State Transfer
Task and group snapshots written to IPC directory for container access:
```typescript
writeTasksSnapshot(group.folder, isMain, tasks)
writeGroupsSnapshot(group.folder, isMain, availableGroups)
```

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

## Database Schema

```
chats              - Chat metadata (jid, name, last_message_time, channel, is_group)
messages           - Full message content (id, chat_jid, sender, content, timestamp, is_from_me, is_bot_message)
scheduled_tasks    - Task definitions (id, group_folder, chat_jid, prompt, schedule_type, schedule_value, next_run, status)
task_run_logs      - Task execution history (task_id, run_at, duration_ms, status, result, error)
router_state      - Key-value state (last_timestamp, last_agent_timestamp)
sessions           - Group -> session_id mapping (group_folder, session_id)
registered_groups  - Group configuration (jid, name, folder, trigger_pattern, is_main, requires_trigger)
```

## IPC Protocol Summary

### Host -> Container

| File | Type | Purpose |
|------|------|---------|
| `stdin` | JSON | Initial ContainerInput |
| `data/ipc/{group}/input/{n}.json` | `{type:'message', text}` | Follow-up messages |
| `data/ipc/{group}/input/_close` | sentinel | Signal container exit |

### Container -> Host

| Channel | Format | Purpose |
|---------|--------|---------|
| `stdout` | `---NANOCLAW_OUTPUT_START---\n{JSON}\n---NANOCLAW_OUTPUT_END---` | Streaming results |
| `data/ipc/{group}/tasks/{n}.json` | IPC command | Task CRUD, group management |
| `data/ipc/{group}/messages/{n}.json` | `{type:'message', chatJid, text}` | Outbound messaging |

### IPC Commands (Container → Host via tasks directory)

- `schedule_task` - Create scheduled task
- `pause_task` / `resume_task` / `cancel_task` / `update_task` - Task lifecycle
- `refresh_groups` - Trigger group metadata sync
- `register_group` - Register new group (main only)
