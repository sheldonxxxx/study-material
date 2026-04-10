# NanoClaw Design Patterns

**Project:** NanoClaw - Personal Claude assistant
**Analysis Date:** 2026-03-26
**Source:** `/Users/sheldon/Documents/claw/reference/nanoclaw`

Design patterns identified in the NanoClaw codebase with specific file references and code evidence.

---

## Pattern 1: Registry/Factory (Self-Registering Channels)

**Problem:** How to add new channels without modifying central registration code.

**Solution:** Channels self-register via side-effect imports. Each channel module calls `registerChannel()` when imported, adding itself to a module-level registry.

### Implementation

**Registry Map:**
```typescript
// src/channels/registry.ts:16-20
const registry = new Map<string, ChannelFactory>();

export function registerChannel(name: string, factory: ChannelFactory): void {
  registry.set(name, factory);
}
```

**Self-Registration via Import:**
```typescript
// src/channels/index.js
// Empty file - imports trigger side-effect registration
import './whatsapp/index.js';  // registers itself
import './telegram/index.js';  // registers itself
```

**Factory Type:**
```typescript
// src/channels/registry.ts:14
export type ChannelFactory = (opts: ChannelOpts) => Channel | null;
```

### Usage in Orchestrator

```typescript
// src/index.ts:16-20
import './channels/index.js';  // triggers all channel registrations
import { getChannelFactory, getRegisteredChannelNames } from './channels/registry.js';

// Later, channels are instantiated:
const factory = getChannelFactory(name);
if (factory) {
  const channel = factory({ onMessage, onChatMetadata, registeredGroups });
  channels.push(channel);
}
```

### Related Patterns

- **Strategy Pattern**: `ChannelFactory` allows different channel implementations with the same interface
- **Plugin Architecture**: Adding a new channel = create directory + add import line

---

## Pattern 2: Strategy Pattern (Channel Abstraction)

**Problem:** Multiple messaging platforms (WhatsApp, Telegram, Slack, Discord, Gmail) with different APIs but same semantics.

**Solution:** `Channel` interface defines common operations; each platform implements the interface.

### Interface

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

### Channel Router (Strategy Selection)

```typescript
// src/router.ts:37-44
export function routeOutbound(
  channels: Channel[],
  jid: string,
  text: string,
): Promise<void> {
  const channel = channels.find((c) => c.ownsJid(jid) && c.isConnected());
  if (!channel) throw new Error(`No channel for JID: ${jid}`);
  return channel.sendMessage(jid, text);
}
```

---

## Pattern 3: Observer Pattern (GroupQueue Callbacks)

**Problem:** GroupQueue manages container lifecycle but needs to notify the orchestrator when messages should be processed.

**Solution:** Callback registration allows the orchestrator to provide a processing function that GroupQueue calls.

### Implementation

```typescript
// src/group-queue.ts:34-36
private processMessagesFn: ((groupJid: string) => Promise<boolean>) | null = null;

setProcessMessagesFn(fn: (groupJid: string) => Promise<boolean>): void {
  this.processMessagesFn = fn;
}
```

```typescript
// src/group-queue.ts:212-220
try {
  if (this.processMessagesFn) {
    const success = await this.processMessagesFn(groupJid);
    if (success) {
      state.retryCount = 0;
    } else {
      this.scheduleRetry(groupJid, state);
    }
  }
}
```

### Usage

```typescript
// src/index.ts
queue.setProcessMessagesFn(async (groupJid: string) => {
  // ... process messages for this group
  return true;
});
```

---

## Pattern 4: AsyncIterable/Stream Pattern (MessageStream)

**Problem:** Streaming messages from IPC files to the Claude SDK which expects an async iterable.

**Solution:** `MessageStream` class implements `AsyncIterable` with push-based message injection.

### Evidence

```typescript
// Container-side code uses AsyncIterable:
// drainIpcInput() returns accumulated message texts
// piped into MessageStream (AsyncIterable)
```

**Reference:** `container/agent-runner/src/index.ts` (MessageStream class with `async *[Symbol.asyncIterator]()`)

---

## Pattern 5: Circuit Breaker Pattern (Retry with Exponential Backoff)

**Problem:** Container failures should not cause immediate retry loops that hammer resources.

**Solution:** Exponential backoff increases delay between retries, with a maximum retry count.

### Implementation

```typescript
// src/group-queue.ts:14-15
const MAX_RETRIES = 5;
const BASE_RETRY_MS = 5000;

// src/group-queue.ts:263-284
private scheduleRetry(groupJid: string, state: GroupState): void {
  state.retryCount++;
  if (state.retryCount > MAX_RETRIES) {
    logger.error(
      { groupJid, retryCount: state.retryCount },
      'Max retries exceeded, dropping messages (will retry on next incoming message)',
    );
    state.retryCount = 0;
    return;
  }

  const delayMs = BASE_RETRY_MS * Math.pow(2, state.retryCount - 1);
  logger.info(
    { groupJid, retryCount: state.retryCount, delayMs },
    'Scheduling retry with backoff',
  );
  setTimeout(() => {
    if (!this.shuttingDown) {
      this.enqueueMessageCheck(groupJid);
    }
  }, delayMs);
}
```

**Retry Schedule:** 5s, 10s, 20s, 40s, 80s (exponential), then drop

---

## Pattern 6: Leased Concurrency Pattern (Bounded Container Pool)

**Problem:** Running unlimited containers could exhaust system resources.

**Solution:** `MAX_CONCURRENT_CONTAINERS` limits active containers; waiting groups are queued.

### Implementation

```typescript
// src/config.ts:58-61
export const MAX_CONCURRENT_CONTAINERS = Math.max(
  1,
  parseInt(process.env.MAX_CONCURRENT_CONTAINERS || '5', 10) || 5,
);

// src/group-queue.ts:73-83
if (this.activeCount >= MAX_CONCURRENT_CONTAINERS) {
  state.pendingMessages = true;
  if (!this.waitingGroups.includes(groupJid)) {
    this.waitingGroups.push(groupJid);
  }
  logger.debug(
    { groupJid, activeCount: this.activeCount },
    'At concurrency limit, message queued',
  );
  return;
}
```

### Draining Waiting Groups

```typescript
// src/group-queue.ts:318-344
private drainWaiting(): void {
  while (
    this.waitingGroups.length > 0 &&
    this.activeCount < MAX_CONCURRENT_CONTAINERS
  ) {
    const nextJid = this.waitingGroups.shift()!;
    const state = this.getGroup(nextJid);

    // Prioritize tasks over messages
    if (state.pendingTasks.length > 0) {
      const task = state.pendingTasks.shift()!;
      this.runTask(nextJid, task);
    } else if (state.pendingMessages) {
      this.runForGroup(nextJid, 'drain');
    }
  }
}
```

---

## Pattern 7: Snapshot/State Transfer Pattern (Tasks and Groups IPC)

**Problem:** Containers need access to task definitions and group metadata at startup.

**Solution:** Host writes current state to JSON files in the container's mount namespace before agent starts.

### Implementation

```typescript
// src/container-runner.ts:673-698
export function writeTasksSnapshot(
  groupFolder: string,
  isMain: boolean,
  tasks: Array<{...}>,
): void {
  const groupIpcDir = resolveGroupIpcPath(groupFolder);
  fs.mkdirSync(groupIpcDir, { recursive: true });

  // Main sees all tasks, others only see their own
  const filteredTasks = isMain
    ? tasks
    : tasks.filter((t) => t.groupFolder === groupFolder);

  const tasksFile = path.join(groupIpcDir, 'current_tasks.json');
  fs.writeFileSync(tasksFile, JSON.stringify(filteredTasks, null, 2));
}
```

```typescript
// src/container-runner.ts:700-736
export function writeGroupsSnapshot(
  groupFolder: string,
  isMain: boolean,
  groups: AvailableGroup[],
  _registeredJids: Set<string>,
): void {
  const groupIpcDir = resolveGroupIpcPath(groupFolder);

  // Main sees all groups; others see nothing (they can't activate groups)
  const visibleGroups = isMain ? groups : [];

  const groupsFile = path.join(groupIpcDir, 'available_groups.json');
  fs.writeFileSync(groupsFile, JSON.stringify({
    groups: visibleGroups,
    lastSync: new Date().toISOString(),
  }, null, 2));
}
```

---

## Pattern 8: Repository Pattern (Database Abstraction)

**Problem:** Direct SQL in business logic creates coupling and duplication.

**Solution:** `db.ts` provides a clean API for all database operations with schema management.

### Schema Creation

```typescript
// src/db.ts:17-85
function createSchema(database: Database.Database): void {
  database.exec(`
    CREATE TABLE IF NOT EXISTS chats (...)
    CREATE TABLE IF NOT EXISTS messages (...)
    CREATE TABLE IF NOT EXISTS scheduled_tasks (...)
    CREATE TABLE IF NOT EXISTS task_run_logs (...)
    CREATE TABLE IF NOT EXISTS router_state (...)
    CREATE TABLE IF NOT EXISTS sessions (...)
    CREATE TABLE IF NOT EXISTS registered_groups (...)
  `);
}
```

### Migration Support

```typescript
// src/db.ts:87-100
// Add context_mode column if it doesn't exist (migration for existing DBs)
try {
  database.exec(
    `ALTER TABLE scheduled_tasks ADD COLUMN context_mode TEXT DEFAULT 'isolated'`,
  );
} catch {
  /* column already exists */
}

// Add script column if it doesn't exist (migration for existing DBs)
try {
  database.exec(`ALTER TABLE scheduled_tasks ADD COLUMN script TEXT`);
} catch {
  /* column already exists */
}
```

---

## Pattern 9: Dependency Injection (Module Dependencies)

**Problem:** Core modules like IPC and Scheduler need access to orchestrator functionality.

**Solution:** Dependency objects are passed explicitly rather than imported directly.

### IpcDeps Interface

```typescript
// src/ipc.ts:13-26
export interface IpcDeps {
  sendMessage: (jid: string, text: string) => Promise<void>;
  registeredGroups: () => Record<string, RegisteredGroup>;
  registerGroup: (jid: string, group: RegisteredGroup) => void;
  syncGroups: (force: boolean) => Promise<void>;
  getAvailableGroups: () => AvailableGroup[];
  writeGroupsSnapshot: (...) => void;
  onTasksChanged: () => void;
}
```

```typescript
// src/ipc.ts:30
export function startIpcWatcher(deps: IpcDeps): void {
  // Uses deps.sendMessage, deps.registeredGroups, etc.
}
```

### SchedulerDependencies Interface

```typescript
// src/task-scheduler.ts:65-76
export interface SchedulerDependencies {
  registeredGroups: () => Record<string, RegisteredGroup>;
  getSessions: () => Record<string, string>;
  queue: GroupQueue;
  onProcess: (groupJid: string, proc: ChildProcess, containerName: string, groupFolder: string) => void;
  sendMessage: (jid: string, text: string) => Promise<void>;
}
```

---

## Pattern 10: Polling Pattern (IPC Watcher and Message Loop)

**Problem:** Event-driven systems need to detect changes without blocking.

**Solution:** Polling loops with configurable intervals check for new work.

### IPC Polling

```typescript
// src/ipc.ts:40-153
const processIpcFiles = async () => {
  // Scan all group IPC directories
  const groupFolders = fs.readdirSync(ipcBaseDir).filter(...);

  for (const sourceGroup of groupFolders) {
    // Process message files and task files
  }

  setTimeout(processIpcFiles, IPC_POLL_INTERVAL);  // 1000ms
};
```

### Message Polling

```typescript
// src/index.ts - message loop
const POLL_INTERVAL = 2000;  // src/config.ts:20

// Polling function runs every POLL_INTERVAL ms
// Checks SQLite for new messages, enqueues for processing
```

### Scheduler Polling

```typescript
// src/config.ts:21
export const SCHEDULER_POLL_INTERVAL = 60000;  // 60 seconds

// task-scheduler.ts runs getDueTasks() on this interval
```

---

## Pattern 11: Sentinel File Pattern (Graceful Shutdown)

**Problem:** How to signal the container to stop accepting new work gracefully.

**Solution:** A sentinel file (`_close`) in the IPC input directory tells the container to exit its processing loop.

### Implementation

```typescript
// src/group-queue.ts:183-194
closeStdin(groupJid: string): void {
  const state = this.getGroup(groupJid);
  if (!state.active || !state.groupFolder) return;

  const inputDir = path.join(DATA_DIR, 'ipc', state.groupFolder, 'input');
  try {
    fs.mkdirSync(inputDir, { recursive: true });
    fs.writeFileSync(path.join(inputDir, '_close'), '');
  } catch {
    // ignore
  }
}
```

### Container Detection

```typescript
// container/agent-runner/src/index.ts
// Container polls for _close file and exits loop when found
```

---

## Pattern 12: Mount Security Pattern (Allowlist Validation)

**Problem:** User-configured additional mounts could expose sensitive host directories.

**Solution:** Mount paths are validated against an external allowlist file stored outside the project directory.

### Validation

```typescript
// src/mount-security.ts
export function validateAdditionalMounts(
  mounts: AdditionalMount[],
  groupName: string,
  isMain: boolean,
): VolumeMount[] {
  // Validated against ~/.config/nanoclaw/mount-allowlist.json
  // Blocked patterns: .ssh, .gnupg, etc.
  // nonMainReadOnly enforcement for non-main groups
}
```

### Allowlist Structure

```typescript
// src/types.ts:12-28
export interface MountAllowlist {
  allowedRoots: AllowedRoot[];     // Directories that can be mounted
  blockedPatterns: string[];       // Patterns to never mount (e.g., ".ssh")
  nonMainReadOnly: boolean;        // Force read-only for non-main groups
}
```

---

## Pattern 13: Work Queue Pattern (Pending Tasks/Messages)

**Problem:** Multiple work items for the same group need to be queued and processed in order.

**Solution:** GroupState holds arrays of pending tasks and messages, processed in FIFO order.

### Implementation

```typescript
// src/group-queue.ts:17-28
interface GroupState {
  active: boolean;
  idleWaiting: boolean;
  isTaskContainer: boolean;
  runningTaskId: string | null;
  pendingMessages: boolean;
  pendingTasks: QueuedTask[];  // Queue of pending work
  process: ChildProcess | null;
  containerName: string | null;
  groupFolder: string | null;
  retryCount: number;
}

// src/group-queue.ts:286-316 (drainGroup)
private drainGroup(groupJid: string): void {
  // Tasks first (they won't be re-discovered from SQLite like messages)
  if (state.pendingTasks.length > 0) {
    const task = state.pendingTasks.shift()!;
    this.runTask(groupJid, task);
    return;
  }

  // Then pending messages
  if (state.pendingMessages) {
    this.runForGroup(groupJid, 'drain');
    return;
  }
}
```

---

## Pattern 14: Builder Pattern (Container Arguments)

**Problem:** Docker run arguments are complex with many conditional mounts and environment variables.

**Solution:** Arguments are built incrementally based on configuration.

### Implementation

```typescript
// src/container-runner.ts:226-275
async function buildContainerArgs(
  mounts: VolumeMount[],
  containerName: string,
  agentIdentifier?: string,
): Promise<string[]> {
  const args: string[] = ['run', '-i', '--rm', '--name', containerName];

  // Add timezone
  args.push('-e', `TZ=${TIMEZONE}`);

  // OneCLI gateway config
  await onecli.applyContainerConfig(args, {...});

  // Host gateway args
  args.push(...hostGatewayArgs());

  // User mapping
  if (hostUid != null && hostUid !== 0 && hostUid !== 1000) {
    args.push('--user', `${hostUid}:${hostGid}`);
    args.push('-e', 'HOME=/home/node');
  }

  // Volume mounts
  for (const mount of mounts) {
    if (mount.readonly) {
      args.push(...readonlyMountArgs(mount.hostPath, mount.containerPath));
    } else {
      args.push('-v', `${mount.hostPath}:${mount.containerPath}`);
    }
  }

  // Image
  args.push(CONTAINER_IMAGE);

  return args;
}
```

---

## Pattern 15: Graceful Shutdown Pattern (No Hard Kills)

**Problem:** Sudden container termination loses work and leaves orphan processes.

**Solution:** On shutdown, GroupQueue detaches containers but does not kill them. They finish naturally via idle timeout or container timeout.

### Implementation

```typescript
// src/group-queue.ts:347-364
async shutdown(_gracePeriodMs: number): Promise<void> {
  this.shuttingDown = true;

  // Count active containers but don't kill them — they'll finish on their own
  // via idle timeout or container timeout. The --rm flag cleans them up on exit.
  // This prevents WhatsApp reconnection restarts from killing working agents.
  const activeContainers: string[] = [];
  for (const [_jid, state] of this.groups) {
    if (state.process && !state.process.killed && state.containerName) {
      activeContainers.push(state.containerName);
    }
  }

  logger.info(
    { activeCount: this.activeCount, detachedContainers: activeContainers },
    'GroupQueue shutting down (containers detached, not killed)',
  );
}
```

---

## Pattern Summary Table

| Pattern | File(s) | Key Evidence |
|---------|---------|--------------|
| Registry/Factory | `src/channels/registry.ts` | `registerChannel()`, side-effect imports |
| Strategy | `src/types.ts`, `src/router.ts` | `Channel` interface, `routeOutbound()` |
| Observer | `src/group-queue.ts` | `setProcessMessagesFn()`, callbacks |
| AsyncIterable | `container/agent-runner/src/index.ts` | `MessageStream` class |
| Circuit Breaker | `src/group-queue.ts` | `scheduleRetry()` with exponential backoff |
| Leased Concurrency | `src/group-queue.ts`, `src/config.ts` | `MAX_CONCURRENT_CONTAINERS` |
| Snapshot/State Transfer | `src/container-runner.ts` | `writeTasksSnapshot()`, `writeGroupsSnapshot()` |
| Repository | `src/db.ts` | Schema management, CRUD operations |
| Dependency Injection | `src/ipc.ts`, `src/task-scheduler.ts` | `IpcDeps`, `SchedulerDependencies` interfaces |
| Polling | `src/ipc.ts`, `src/index.ts` | `setTimeout(processIpcFiles, IPC_POLL_INTERVAL)` |
| Sentinel File | `src/group-queue.ts` | `_close` sentinel file |
| Mount Security | `src/mount-security.ts`, `src/types.ts` | Allowlist validation |
| Work Queue | `src/group-queue.ts` | `pendingTasks[]`, `pendingMessages` |
| Builder | `src/container-runner.ts` | `buildContainerArgs()` |
| Graceful Shutdown | `src/group-queue.ts` | `shutdown()` with detach, not kill |
