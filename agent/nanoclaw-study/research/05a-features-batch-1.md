# NanoClaw Features Deep Dive: Batch 1 (Features 1-3)

**Project:** NanoClaw - Personal Claude assistant daemon
**Repo:** `/Users/sheldon/Documents/claw/reference/nanoclaw`
**Date:** 2026-03-26
**Features Covered:** Container Isolation, Multi-Channel Messaging, Group-Based Context Isolation

---

## Feature 1: Container Isolation

### Overview

Agents run in isolated Linux containers with filesystem isolation. The container runtime abstraction (`src/container-runtime.ts`) supports Docker on macOS/Linux, with an Apple Container conversion skill available. Only explicitly mounted directories are accessible inside containers.

### Architecture

**Two-layer abstraction:**

1. **`src/container-runtime.ts`** - Runtime-agnostic interface
   - `CONTAINER_RUNTIME_BIN` - Currently hardcoded to `'docker'`
   - `hostGatewayArgs()` - Returns `['--add-host=host.docker.internal:host-gateway']` on Linux (not needed on macOS where Docker Desktop provides `host.docker.internal` natively)
   - `readonlyMountArgs()` - Returns `-v hostPath:containerPath:ro` bind mount syntax
   - `stopContainer(name)` - Generates `docker stop -t 1 <name>`
   - `ensureContainerRuntimeRunning()` - Runs `docker info` with 10s timeout, throws fatal error if fails
   - `cleanupOrphans()` - Stops any leftover `nanoclaw-*` containers from crashed runs

2. **`src/container-runner.ts`** - High-level orchestration
   - Builds per-group volume mount configurations
   - Spawns containers with `spawn(CONTAINER_RUNTIME_BIN, containerArgs, { stdio: ['pipe', 'pipe', 'pipe'] })`
   - Streams JSON input via stdin, parses JSON output between sentinel markers

### Volume Mount Strategy

The `buildVolumeMounts()` function constructs mounts based on whether the group is `main` or not:

```typescript
// Main group: project root read-only + group folder writable + IPC + sessions
// Non-main groups: own folder writable + global memory read-only + IPC + sessions
```

**Key mounts for all groups:**

| Mount | Purpose | Readonly |
|-------|---------|----------|
| `/workspace/group` | Per-group working directory | No |
| `/workspace/global` | Shared global memory (non-main only) | Yes |
| `/workspace/ipc` | Inter-process communication | No |
| `/home/node/.claude` | Per-group sessions + skills | No |
| `/app/src` | Per-group agent-runner source | No |

**Main group additional mounts:**

```typescript
// Project root read-only (prevents agent from modifying src/, dist/, package.json)
{ hostPath: projectRoot, containerPath: '/workspace/project', readonly: true }

// Shadow .env with /dev/null (credentials injected by OneCLI gateway instead)
{ hostPath: '/dev/null', containerPath: '/workspace/project/.env', readonly: true }
```

### Session Isolation

Each group gets its own `.claude/` directory at `data/sessions/{group-folder}/.claude`:

```typescript
const groupSessionsDir = path.join(DATA_DIR, 'sessions', group.folder, '.claude');
fs.mkdirSync(groupSessionsDir, { recursive: true });

// settings.json enables agent teams and memory
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1",
    "CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD": "1",
    "CLAUDE_CODE_DISABLE_AUTO_MEMORY": "0"
  }
}
```

### IPC Namespace Isolation

Each group gets its own IPC directory at `data/ipc/{group-folder}/` with subdirectories:
- `messages/` - outbound messages written by container, read by host
- `tasks/` - scheduled task IPC files
- `input/` - follow-up messages from host to container

```typescript
const groupIpcDir = resolveGroupIpcPath(group.folder);
fs.mkdirSync(path.join(groupIpcDir, 'messages'), { recursive: true });
fs.mkdirSync(path.join(groupIpcDir, 'tasks'), { recursive: true });
fs.mkdirSync(path.join(groupIpcDir, 'input'), { recursive: true });
```

### Additional Mounts Security

Additional mounts (host directories the agent can access) are validated against a tamper-proof allowlist:

```typescript
// Allowlist stored at ~/.config/nanoclaw/mount-allowlist.json (NOT in project, NOT mounted)
export const MOUNT_ALLOWLIST_PATH = path.join(HOME_DIR, '.config', 'nanoclaw', 'mount-allowlist.json');

if (group.containerConfig?.additionalMounts) {
  const validatedMounts = validateAdditionalMounts(
    group.containerConfig.additionalMounts,
    group.name,
    isMain,
  );
  mounts.push(...validatedMounts);
}
```

The allowlist structure:
```typescript
interface MountAllowlist {
  allowedRoots: Array<{ path: string; allowReadWrite: boolean; description?: string }>;
  blockedPatterns: string[];  // e.g., [".ssh", ".gnupg"]
  nonMainReadOnly: boolean;   // Force read-only for non-main groups
}
```

### Container Lifecycle

1. **Spawn**: `docker run -i --rm --name nanoclaw-{group}-{timestamp}`
2. **Input**: JSON written to stdin containing `{ prompt, sessionId, groupFolder, chatJid, isMain }`
3. **Execute**: Agent processes, streams output via `OUTPUT_START_MARKER` / `OUTPUT_END_MARKER` pairs
4. **Timeout**: Hard timeout (default 30min) with graceful shutdown (15s grace via `docker stop -t 1`)
5. **Cleanup**: `--rm` flag ensures container removed on exit

### Output Streaming

Sentinel markers enable streaming parsing before container exit:

```typescript
const OUTPUT_START_MARKER = '---NANOCLAW_OUTPUT_START---';
const OUTPUT_END_MARKER = '---NANOCLAW_OUTPUT_END---';

// Stream-parse for output markers
parseBuffer += chunk;
let startIdx: number;
while ((startIdx = parseBuffer.indexOf(OUTPUT_START_MARKER)) !== -1) {
  const endIdx = parseBuffer.indexOf(OUTPUT_END_MARKER, startIdx);
  if (endIdx === -1) break;
  const jsonStr = parseBuffer.slice(startIdx + OUTPUT_START_MARKER.length, endIdx).trim();
  const parsed: ContainerOutput = JSON.parse(jsonStr);
  outputChain = outputChain.then(() => onOutput(parsed));
}
```

### Clever Solutions

1. **Shadow .env with /dev/null**: Main group's project root mount would expose `.env` with real credentials. Shadowing with `/dev/null` nullifies this without changing the mount path.
2. **Per-group agent-runner source copy**: Agent can customize its runner without affecting other groups or requiring rebuild:
   ```typescript
   const needsCopy = !fs.existsSync(groupAgentRunnerDir) ||
     !fs.existsSync(cachedIndex) ||
     (fs.existsSync(srcIndex) && fs.statSync(srcIndex).mtimeMs > fs.statSync(cachedIndex).mtimeMs);
   ```
3. **Graceful timeout distinction**: If container timed out but had streaming output, it's treated as success (idle cleanup after response sent). Only timeouts with no output are errors.

### Technical Debt / Concerns

1. **Docker hardcoded**: `CONTAINER_RUNTIME_BIN = 'docker'` with no abstraction layer for Podman or other runtimes. The skill `convert-to-apple-container` suggests Apple Container support exists but is a separate skill.
2. **Output truncation**: 10MB max stdout/stderr with no streaming to disk for large outputs.
3. **No resource limits**: No CPU/memory constraints on containers (Docker's `--memory`, `--cpus` flags not used).

---

## Feature 2: Multi-Channel Messaging

### Overview

A unified messaging interface for WhatsApp, Telegram, Discord, Slack, and Gmail. Channels are self-registering modules added via skill branches (`/add-telegram`, `/add-whatsapp`, etc.). The system polls all channels in the main loop and routes messages to containers.

### Self-Registration Architecture

**Channel registry** (`src/channels/registry.ts`):

```typescript
const registry = new Map<string, ChannelFactory>();

export function registerChannel(name: string, factory: ChannelFactory): void {
  registry.set(name, factory);
}

export function getChannelFactory(name: string): ChannelFactory | undefined {
  return registry.get(name);
}
```

**Barrel file** (`src/channels/index.ts`) triggers self-registration via side-effect imports:

```typescript
// discord
// gmail
// slack
// telegram
// whatsapp
```

Each channel module (e.g., `src/channels/telegram.ts`) exports a class that calls `registerChannel('telegram', factory)` when imported. The barrel file import in `src/index.ts` ensures all present channels register at startup:

```typescript
import './channels/index.js';  // Side-effect imports trigger registration
```

**Channel factory pattern:**

```typescript
type ChannelFactory = (opts: ChannelOpts) => Channel | null;

interface ChannelOpts {
  onMessage: OnInboundMessage;
  onChatMetadata: OnChatMetadata;
  registeredGroups: () => Record<string, RegisteredGroup>;
}

interface Channel {
  name: string;
  connect(): Promise<void>;
  sendMessage(jid: string, text: string): Promise<void>;
  isConnected(): boolean;
  ownsJid(jid: string): boolean;
  disconnect(): Promise<void>;
  setTyping?(jid: string, isTyping: boolean): Promise<void>;
  syncGroups?(force: boolean): Promise<void>;
}
```

### Skill-Based Channel Installation

Channels are added via skill branches that merge code into the repo. Example: `add-telegram/SKILL.md`:

```bash
git remote add telegram https://github.com/qwibitai/nanoclaw-telegram.git
git fetch telegram main
git merge telegram/main
```

The merge brings:
- `src/channels/telegram.ts` - Channel implementation
- `src/channels/telegram.test.ts` - Unit tests
- `import './telegram.js'` appended to `src/channels/index.ts`
- `grammy` npm dependency
- `TELEGRAM_BOT_TOKEN` in `.env.example`

**Channels auto-enable when credentials are present** - no extra configuration after merge.

### Message Flow

1. **Channel polls** its platform (polling interval handled per-channel)
2. **New messages** stored in SQLite via `storeMessage()` callback
3. **Main loop** (`startMessageLoop()`) polls SQLite for new messages every 2 seconds
4. **Trigger check**: Non-main groups require trigger word (e.g., `@Andy`) before acting
5. **Group queue**: Messages queued if container at concurrency limit
6. **Container run**: Agent invoked, streaming output sent back via channel
7. **Outbound formatting**: `<internal>` tags stripped, result sent via `channel.sendMessage()`

### Channel Callbacks

Two callbacks shared across all channels:

```typescript
const channelOpts = {
  onMessage: (chatJid: string, msg: NewMessage) => {
    // /remote-control command interception
    // Sender allowlist filtering
    storeMessage(msg);  // Persist to SQLite
  },
  onChatMetadata: (chatJid, timestamp, name?, channel?, isGroup?) => {
    storeChatMetadata(chatJid, timestamp, name, channel, isGroup);
  },
  registeredGroups: () => registeredGroups,
};
```

### JID Naming Convention

JIDs identify chats across channels using channel-specific prefixes:
- Telegram: `tg:123456789` (个人) or `tg:-1001234567890` (groups)
- WhatsApp: `wa:123456789` (structure varies by WhatsApp version)
- Slack: `slack:CHANNEL_ID`
- Discord: `discord:CHANNEL_ID`
- Gmail: `gmail:thread_id` or similar

`findChannel()` routes JIDs to their owning channel:

```typescript
export function findChannel(channels: Channel[], jid: string): Channel | undefined {
  return channels.find((c) => c.ownsJid(jid));
}
```

### Skills Available for Channels

- `add-whatsapp` - WhatsApp integration
- `add-telegram` - Telegram integration (with optional swarm support)
- `add-discord` - Discord integration
- `add-slack` - Slack integration
- `add-gmail` - Gmail integration
- `add-telegram-swarm` - Multi-agent Telegram support
- `channel-formatting` - Normalizes message formatting across channels

### Clever Solutions

1. **Self-registration**: Adding a new channel only requires merging a skill branch - no code changes to `index.ts` or registry
2. **Trigger accumulation**: Messages without trigger word are stored but not acted on; they become context when a triggered message arrives:
   ```typescript
   // Pull all messages since lastAgentTimestamp so non-trigger
   // context that accumulated between triggers is included.
   const allPending = getMessagesSince(chatJid, lastAgentTimestamp[chatJid] || '', ASSISTANT_NAME);
   ```
3. **Cursor rollback on error**: If agent fails before sending output, message cursor rolls back to retry on next cycle:
   ```typescript
   if (outputSentToUser) return true;  // Don't rollback - user got response
   lastAgentTimestamp[chatJid] = previousCursor;  // Rollback for retry
   ```

### Technical Debt / Concerns

1. **No channel-specific code in core**: Channels live in separate skill repos (`qwibitai/nanoclaw-telegram`, etc.) - increases maintenance burden and cross-repo coordination
2. **Polling-based**: All channels use polling (SQLite every 2s + channel-specific polling). No webhooks for real-time delivery.
3. **Bare barrel file**: `src/channels/index.ts` has commented imports - future skills must uncomment or the pattern breaks

---

## Feature 3: Group-Based Context Isolation

### Overview

Each conversation group has its own isolated context: a `CLAUDE.md` memory file, a filesystem mount, a container sandbox, and an IPC namespace. This prevents cross-group information leakage and resource contention.

### Group Folder Structure

```
groups/
  global/              # Global memory (read-only for non-main)
    CLAUDE.md
  main/                # Main control group (no trigger required)
    CLAUDE.md
    logs/
  {group-name}/        # Per-registered-group
    CLAUDE.md          # Generated from template on group creation
    logs/
```

### Registration Flow

When a group registers (`registerGroup()` in `src/index.ts`):

```typescript
function registerGroup(jid: string, group: RegisteredGroup): void {
  registeredGroups[jid] = group;
  setRegisteredGroup(jid, group);  // Persist to SQLite

  // Create group folder
  const groupDir = resolveGroupFolderPath(group.folder);
  fs.mkdirSync(path.join(groupDir, 'logs'), { recursive: true });

  // Copy CLAUDE.md template
  const groupMdFile = path.join(groupDir, 'CLAUDE.md');
  if (!fs.existsSync(groupMdFile)) {
    const templateFile = path.join(GROUPS_DIR, group.isMain ? 'main' : 'global', 'CLAUDE.md');
    let content = fs.readFileSync(templateFile, 'utf-8');
    if (ASSISTANT_NAME !== 'Andy') {
      content = content.replace(/^# Andy$/m, `# ${ASSISTANT_NAME}`);
      content = content.replace(/You are Andy/g, `You are ${ASSISTANT_NAME}`);
    }
    fs.writeFileSync(groupMdFile, content);
  }
}
```

### Per-Group IPC Namespacing

IPC directory per group: `data/ipc/{group-folder}/`

The IPC watcher (`src/ipc.ts`) scans all group directories and processes messages/tasks from each:

```typescript
const ipcBaseDir = path.join(DATA_DIR, 'ipc');
groupFolders = fs.readdirSync(ipcBaseDir).filter(f => {
  const stat = fs.statSync(path.join(ipcBaseDir, f));
  return stat.isDirectory() && f !== 'errors';
});

for (const sourceGroup of groupFolders) {
  const isMain = folderIsMain.get(sourceGroup) === true;
  // Process messages from data/ipc/{sourceGroup}/messages/
  // Process tasks from data/ipc/{sourceGroup}/tasks/
}
```

### Authorization Model

IPC operations verify identity via directory path (sourceGroup) and isMain flag derived from registered groups:

```typescript
// Authorization: non-main groups can only schedule for themselves
if (!isMain && targetFolder !== sourceGroup) {
  logger.warn({ sourceGroup, targetFolder }, 'Unauthorized schedule_task attempt blocked');
  break;
}

// IPC message authorization: main can send anywhere, others only to themselves
const targetGroup = registeredGroups[data.chatJid];
if (isMain || (targetGroup && targetGroup.folder === sourceGroup)) {
  await deps.sendMessage(data.chatJid, data.text);
} else {
  logger.warn({ chatJid: data.chatJid, sourceGroup }, 'Unauthorized IPC message attempt blocked');
}
```

### Group Queue Concurrency Control

`src/group-queue.ts` implements per-group concurrency limiting:

```typescript
const MAX_CONCURRENT_CONTAINERS = 5;  // Configurable via env

interface GroupState {
  active: boolean;
  idleWaiting: boolean;
  isTaskContainer: boolean;
  runningTaskId: string | null;
  pendingMessages: boolean;
  pendingTasks: QueuedTask[];
  process: ChildProcess | null;
  containerName: string | null;
  groupFolder: string | null;
  retryCount: number;
}
```

**Queue behavior:**
- Global concurrency limit shared across all groups
- Per-group queues for messages and tasks
- Tasks prioritized over messages in drain
- Idle containers kept alive for `IDLE_TIMEOUT` (default 30min) to receive follow-ups
- Preemptive stdin close (`_close` sentinel) when new task arrives for idle container

### Idle Container Reuse

When a message arrives for a group with an idle container:

```typescript
// queue.sendMessage() - sends follow-up via IPC file
sendMessage(groupJid: string, text: string): boolean {
  const state = this.getGroup(groupJid);
  if (!state.active || !state.groupFolder || state.isTaskContainer) return false;
  state.idleWaiting = false;  // Agent is about to receive work

  const inputDir = path.join(DATA_DIR, 'ipc', state.groupFolder, 'input');
  fs.writeFileSync(path.join(inputDir, `${Date.now()}.json`), JSON.stringify({ type: 'message', text }));
  return true;
}
```

### Path Validation

Group folders validated against strict pattern to prevent path traversal:

```typescript
const GROUP_FOLDER_PATTERN = /^[A-Za-z0-9][A-Za-z0-9_-]{0,63}$/;
const RESERVED_FOLDERS = new Set(['global']);

export function isValidGroupFolder(folder: string): boolean {
  if (!folder) return false;
  if (folder !== folder.trim()) return false;
  if (!GROUP_FOLDER_PATTERN.test(folder)) return false;
  if (folder.includes('/') || folder.includes('\\')) return false;
  if (folder.includes('..')) return false;
  if (RESERVED_FOLDERS.has(folder.toLowerCase())) return false;
  return true;
}
```

Path resolution uses `ensureWithinBase()` to prevent directory escape:

```typescript
function ensureWithinBase(baseDir: string, resolvedPath: string): void {
  const rel = path.relative(baseDir, resolvedPath);
  if (rel.startsWith('..') || path.isAbsolute(rel)) {
    throw new Error(`Path escapes base directory: ${resolvedPath}`);
  }
}
```

### Snapshot System

Containers receive contextual information via filesystem snapshots:

**Tasks snapshot** (`writeTasksSnapshot`):
```typescript
// Main sees all tasks; others only see their own
const filteredTasks = isMain ? tasks : tasks.filter(t => t.groupFolder === groupFolder);
fs.writeFileSync(path.join(groupIpcDir, 'current_tasks.json'), JSON.stringify(filteredTasks));
```

**Groups snapshot** (`writeGroupsSnapshot`):
```typescript
// Main sees all available groups (for activation); others see nothing
const visibleGroups = isMain ? groups : [];
fs.writeFileSync(path.join(groupIpcDir, 'available_groups.json'), JSON.stringify({ groups: visibleGroups }));
```

### Main vs Non-Main Groups

| Aspect | Main Group | Non-Main Groups |
|--------|-----------|-----------------|
| Trigger required | No | Yes (default) |
| Can register groups | Yes | No |
| Can refresh groups | Yes | No |
| Sees all tasks | Yes | No |
| Sees all available groups | Yes | No |
| Project root accessible | Read-only via `/workspace/project` | No |
| Mounts global memory | Yes (read-only) | Yes (read-only) |

### Global Memory

`groups/global/CLAUDE.md` is mounted read-only at `/workspace/global` for all non-main groups. This provides shared context (organization info, team knowledge) without group-specific memory leaking between groups.

### Clever Solutions

1. **Identity via directory path**: IPC authorization uses the directory containing the file, not the file contents - prevents tampering with IPC files to spoof identity
2. **Template substitution**: On group creation, `{{ASSISTANT_NAME}}` is replaced in CLAUDE.md template, so custom names work without modifying templates
3. **Exponential backoff retry**: Message processing failures use exponential backoff:
   ```typescript
   const delayMs = BASE_RETRY_MS * Math.pow(2, state.retryCount - 1);
   ```
4. **Cursor rollback isolation**: Each group has independent `lastAgentTimestamp` - one group's error doesn't affect others

### Technical Debt / Concerns

1. **No group deletion**: No `unregisterGroup` IPC type or cleanup of group folders/sessions
2. **Race condition in recovery**: `recoverPendingMessages()` runs at startup but uses `lastAgentTimestamp` which may not reflect in-progress processing from a crashed instance
3. **Global concurrency limit**: All groups share `MAX_CONCURRENT_CONTAINERS` regardless of group priority or size
4. **No per-group resource limits**: CPU/memory not isolated per group, only container-level

---

## Cross-Feature Interactions

### Container Isolation + Group Context

Containers are spawned per-message-batch, not per-group. The same container may handle multiple sequential message batches for a group (kept alive via `IDLE_TIMEOUT`). Each container's filesystem view is isolated via volume mounts - no container can access another group's data.

```
Container for group-A:
  /workspace/group    -> groups/main/ or groups/telegram_main/
  /workspace/global   -> groups/global/
  /workspace/ipc      -> data/ipc/{group-folder}/
  /home/node/.claude  -> data/sessions/{group-folder}/.claude/

Container for group-B:
  /workspace/group    -> groups/other_group/
  /workspace/global   -> groups/global/
  /workspace/ipc      -> data/ipc/{other_group}/
  /home/node/.claude  -> data/sessions/{other_group}/.claude/
```

### Multi-Channel + Group Context

A single group can receive messages from multiple channels (e.g., a Telegram group + a WhatsApp group both mapped to the same `folder`). The JID determines routing, but group context (CLAUDE.md, sessions, IPC) is shared by folder name.

### IPC + Container Isolation

IPC files flow from container to host via mounted directories:
- Container writes response to `/workspace/ipc/messages/{uuid}.json`
- Host IPC watcher reads, sends via channel, deletes file
- Host can write follow-up to `/workspace/ipc/input/{uuid}.json`
- Container reads input file (polled by agent-runner)

The `_close` sentinel in `input/` signals the agent to gracefully terminate.

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `src/container-runtime.ts` | Runtime abstraction (Docker) |
| `src/container-runner.ts` | Container spawning, mount building, output parsing |
| `src/channels/registry.ts` | Channel factory registry |
| `src/channels/index.ts` | Channel self-registration barrel |
| `src/index.ts` | Main orchestrator, channel setup, message loop |
| `src/group-queue.ts` | Per-group concurrency control |
| `src/group-folder.ts` | Path validation and resolution |
| `src/ipc.ts` | IPC watcher, task processing, authorization |
| `src/router.ts` | Message formatting, channel routing |
| `src/types.ts` | Channel, Group, Message interfaces |
| `groups/*/CLAUDE.md` | Per-group memory templates |
| `.claude/skills/add-*/SKILL.md` | Channel installation skills |
