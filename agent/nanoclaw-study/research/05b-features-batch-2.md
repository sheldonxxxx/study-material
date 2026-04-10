# Features 4-6 Deep Dive: Message Orchestration, Scheduled Tasks, Per-Group Queue

**Project:** NanoClaw
**Analysis Date:** 2026-03-26
**Files Analyzed:**
- `src/index.ts` (main orchestrator, ~718 lines)
- `src/router.ts` (message formatting/routing, ~53 lines)
- `src/task-scheduler.ts` (scheduled task execution, ~285 lines)
- `src/group-queue.ts` (concurrency control, ~366 lines)
- `src/config.ts` (configuration constants)
- `src/timezone.ts` (timezone utilities)
- `src/types.ts` (type definitions)

---

## Feature 4: Message Orchestration

### Overview

The message orchestration is the central polling loop that routes messages from all channels to the Claude Agent SDK and back. It is implemented as a single `while(true)` loop in `src/index.ts` that polls all registered channels for new messages.

### Architecture

**Core Components:**
- `startMessageLoop()` - The main polling loop (lines 395-495 in `src/index.ts`)
- `processGroupMessages()` - Processes pending messages for a specific group (lines 196-312)
- `runAgent()` - Invokes the containerized Claude agent (lines 314-393)
- `findChannel()` - Routes JIDs to their owning channel (in `src/router.ts`)

**Flow:**
```
Channels (polling) --> SQLite (store messages) --> Message Loop (pickup)
                                                              |
                                          Format messages as XML context
                                                              |
                                          GroupQueue (concurrency control)
                                                              |
                                          Container (Claude Agent SDK)
                                                              |
                                          Streaming output callback
                                                              |
                                          Channel.sendMessage() (response)
```

### Key Implementation Details

**1. Polling Loop Structure (src/index.ts:404-494)**

```typescript
while (true) {
  try {
    const jids = Object.keys(registeredGroups);
    const { messages, newTimestamp } = getNewMessages(jids, lastTimestamp, ASSISTANT_NAME);

    if (messages.length > 0) {
      lastTimestamp = newTimestamp;
      saveState();

      // Deduplicate by group
      const messagesByGroup = new Map<string, NewMessage[]>();
      for (const msg of messages) {
        // ... grouping logic
      }

      for (const [chatJid, groupMessages] of messagesByGroup) {
        // ... trigger checking, message formatting
        if (queue.sendMessage(chatJid, formatted)) {
          // Piped to active container
        } else {
          // Enqueue for new container
          queue.enqueueMessageCheck(chatJid);
        }
      }
    }
  } catch (err) {
    logger.error({ err }, 'Error in message loop');
  }
  await new Promise((resolve) => setTimeout(resolve, POLL_INTERVAL));
}
```

**Key Design Decisions:**
- **Global message cursor**: `lastTimestamp` advances for ALL messages across ALL groups in one atomic write. This prevents duplicate processing on crash.
- **Per-group agent cursor**: `lastAgentTimestamp[chatJid]` tracks the last message each group's agent has consumed, enabling proper context accumulation between triggers.
- **Trigger filtering**: Non-main groups require the trigger pattern (default `@Andy`) for the assistant to respond. Messages WITHOUT the trigger still accumulate in SQLite and are pulled as context when a trigger message arrives (lines 459-467).

**2. Message Formatting (src/router.ts:13-25)**

Messages are formatted into an XML context block:

```typescript
export function formatMessages(messages: NewMessage[], timezone: string): string {
  const lines = messages.map((m) => {
    const displayTime = formatLocalTime(m.timestamp, timezone);
    return `<message sender="${escapeXml(m.sender_name)}" time="${escapeXml(displayTime)}">${escapeXml(m.content)}</message>`;
  });

  const header = `<context timezone="${escapeXml(timezone)}" />\n`;
  return `${header}<messages>\n${lines.join('\n')}\n</messages>`;
}
```

**3. Active Container Message Piping (src/index.ts:459-487)**

When a message arrives and an active container exists for that group, messages are "piped" directly via filesystem IPC:

```typescript
if (queue.sendMessage(chatJid, formatted)) {
  // Message written to IPC input file for the active container
  lastAgentTimestamp[chatJid] = messagesToSend[messagesToSend.length - 1].timestamp;
  saveState();
  channel.setTyping?.(chatJid, true);  // Show typing indicator
}
```

**4. Idle Timeout (src/index.ts:244-255)**

```typescript
const resetIdleTimer = () => {
  if (idleTimer) clearTimeout(idleTimer);
  idleTimer = setTimeout(() => {
    queue.closeStdin(chatJid);  // Signal container to wind down
  }, IDLE_TIMEOUT);  // Default: 30 minutes
};
```

### Clever Solutions

**1. Context Accumulation Between Triggers**
When a trigger message arrives, the system pulls ALL messages since `lastAgentTimestamp` (line 461-467), not just the trigger message. This means accumulated conversation context between triggers is preserved.

**2. Cursor Rollback on Error (src/index.ts:291-308)**
If the agent fails after sending output to the user, the cursor is NOT rolled back to prevent duplicate responses. Only when no output was sent does it roll back for retry.

**3. Trigger-Only Processing with Context Preservation**
Non-trigger messages are stored in SQLite but don't advance the agent cursor. When a trigger finally arrives, `getMessagesSince()` pulls everything the agent hasn't seen yet.

### Error Handling

- **Polling errors**: Caught individually per iteration, logged, loop continues
- **Agent errors**: Cursor rollback for retry (unless output was sent)
- **Channel send errors**: Logged but don't crash the loop
- **Recovery on startup**: `recoverPendingMessages()` (lines 501-513) re-enqueues unprocessed messages on restart

### Edge Cases

1. **No active container + incoming message**: `queue.sendMessage()` returns false, triggering `queue.enqueueMessageCheck()` to start a new container
2. **Trigger message without trigger prefix in content**: Uses `triggerPattern.test(m.content.trim())` to check content only
3. **Self-messages (`is_from_me`)**: Bypass trigger and allowlist checks
4. **Message loop crash**: Logged as fatal, process exits (line 700-702)

---

## Feature 5: Scheduled Tasks

### Overview

Scheduled tasks are cron-style recurring jobs that run Claude agents automatically and can message results back. Implemented in `src/task-scheduler.ts` with support for cron expressions, interval-based scheduling, and one-time tasks.

### Architecture

**Core Components:**
- `startSchedulerLoop()` - Polling loop for due tasks (lines 245-279)
- `runTask()` - Executes a single scheduled task (lines 78-241)
- `computeNextRun()` - Calculates the next execution time (lines 31-63)
- Database functions: `getDueTasks()`, `getTaskById()`, `updateTaskAfterRun()`, `logTaskRun()`

### Key Implementation Details

**1. Schedule Types (src/types.ts:56-70)**

```typescript
export interface ScheduledTask {
  schedule_type: 'cron' | 'interval' | 'once';
  schedule_value: string;        // cron expr, ms interval, or ignored
  context_mode: 'group' | 'isolated';
  next_run: string | null;        // ISO timestamp
  last_run: string | null;
  last_result: string | null;
  status: 'active' | 'paused' | 'completed';
}
```

**2. Next Run Computation (src/task-scheduler.ts:31-63)**

```typescript
export function computeNextRun(task: ScheduledTask): string | null {
  if (task.schedule_type === 'once') return null;

  const now = Date.now();

  if (task.schedule_type === 'cron') {
    const interval = CronExpressionParser.parse(task.schedule_value, {
      tz: TIMEZONE,
    });
    return interval.next().toISOString();
  }

  if (task.schedule_type === 'interval') {
    const ms = parseInt(task.schedule_value, 10);
    if (!ms || ms <= 0) {
      logger.warn({ taskId: task.id, value: task.schedule_value }, 'Invalid interval value');
      return new Date(now + 60_000).toISOString();  // Safe fallback
    }
    // Anchor to scheduled time, not now, to prevent drift
    let next = new Date(task.next_run!).getTime() + ms;
    while (next <= now) { next += ms; }  // Skip missed intervals
    return new Date(next).toISOString();
  }
  return null;
}
```

**Notable Design: Drift Prevention via Time Anchoring**
For interval tasks, the next run is calculated from `task.next_run` (the original scheduled time), NOT `Date.now()`. A while loop catches up any missed intervals. This prevents cumulative drift where a 5-minute task running every 5 minutes would gradually shift away from the original schedule.

**3. Scheduler Polling Loop (src/task-scheduler.ts:245-279)**

```typescript
export function startSchedulerLoop(deps: SchedulerDependencies): void {
  const loop = async () => {
    try {
      const dueTasks = getDueTasks();
      for (const task of dueTasks) {
        // Re-check status in case it was paused/cancelled
        const currentTask = getTaskById(task.id);
        if (!currentTask || currentTask.status !== 'active') {
          continue;
        }
        deps.queue.enqueueTask(currentTask.chat_jid, currentTask.id, () =>
          runTask(currentTask, deps),
        );
      }
    } catch (err) {
      logger.error({ err }, 'Error in scheduler loop');
    }
    setTimeout(loop, SCHEDULER_POLL_INTERVAL);  // 60 second default
  };
  loop();
}
```

**4. Task Execution (src/task-scheduler.ts:78-241)**

Tasks run through the same `GroupQueue` as messages, but with special handling:

```typescript
const TASK_CLOSE_DELAY_MS = 10000;  // 10 seconds

const scheduleClose = () => {
  if (closeTimer) return;
  closeTimer = setTimeout(() => {
    deps.queue.closeStdin(task.chat_jid);  // Close after result
  }, TASK_CLOSE_DELAY_MS);
};
```

Key differences from message-based containers:
- **Single-turn execution**: Tasks run once and close, unlike message containers that stay alive for streaming
- **Prompt delivery via `runContainerAgent()`**: The `prompt` field becomes the agent's initial prompt
- **Optional script execution**: `task.script` allows running arbitrary scripts
- **Results forwarded to user**: `await deps.sendMessage(task.chat_jid, streamedOutput.result)`

**5. Error Handling and Task State Transitions (src/task-scheduler.ts:86-102)**

```typescript
try {
  groupDir = resolveGroupFolderPath(task.group_folder);
} catch (err) {
  // Stop retry churn for malformed legacy rows
  updateTask(task.id, { status: 'paused' });
  logTaskRun({ task_id: task.id, run_at: new Date().toISOString(), duration_ms: Date.now() - startTime, status: 'error', result: null, error });
  return;
}
```

**6. Context Modes (src/task-scheduler.ts:154-156)**

```typescript
const sessions = deps.getSessions();
const sessionId =
  task.context_mode === 'group' ? sessions[task.group_folder] : undefined;
```

- `'group'`: Uses the group's current conversation session
- `'isolated'`: Creates a fresh session for each task run

### Clever Solutions

**1. Missed Interval Catch-Up**
For interval tasks, if the system was down for a while, the while loop in `computeNextRun()` will iterate until `next > now`, ensuring tasks don't pile up indefinitely.

**2. Prompt Closure with Short Delay**
Tasks close stdin after 10 seconds (`TASK_CLOSE_DELAY_MS`) rather than waiting for `IDLE_TIMEOUT` (30 minutes). This is appropriate because tasks are single-turn.

**3. Task/Message Container Distinction**
The `isTaskContainer` flag in `GroupState` prevents `sendMessage()` from piping to task containers (line 162 in `group-queue.ts`).

### Technical Debt / Concerns

1. **60-second scheduler poll interval**: For cron tasks with minute-level precision, this is acceptable but could miss sub-minute tasks.
2. **No distributed scheduling**: All tasks run in a single Node.js process. If the process restarts, tasks are re-enqueued on startup.
3. **No task deduplication**: If the scheduler loop runs twice before a task completes, `enqueueTask()` has a `runningTaskId` check, but this is per-group, not global.

---

## Feature 6: Per-Group Queue with Concurrency Control

### Overview

Global concurrency limiting ensures resources are shared fairly across groups while preventing overload. Implemented as the `GroupQueue` class in `src/group-queue.ts`.

### Architecture

**Core Class:** `GroupQueue` (lines 30-365 in `src/group-queue.ts`)

**State Tracking:**
```typescript
interface GroupState {
  active: boolean;
  idleWaiting: boolean;           // Container finished but still alive
  isTaskContainer: boolean;        // True for scheduled tasks
  runningTaskId: string | null;    // Prevents double-task execution
  pendingMessages: boolean;
  pendingTasks: QueuedTask[];
  process: ChildProcess | null;    // Reference to container process
  containerName: string | null;
  groupFolder: string | null;
  retryCount: number;
}
```

**Concurrency Limit:** `MAX_CONCURRENT_CONTAINERS` (default: 5, from config)

### Key Implementation Details

**1. Global Concurrency Counter (src/group-queue.ts:32)**

```typescript
private activeCount = 0;  // Shared across all groups
```

This is the key mechanism preventing overload. All groups share the same counter.

**2. Enqueue Logic (src/group-queue.ts:62-88 for messages, 90-130 for tasks)**

```typescript
enqueueMessageCheck(groupJid: string): void {
  const state = this.getGroup(groupJid);

  if (state.active) {
    state.pendingMessages = true;  // Already processing, queue for later
    return;
  }

  if (this.activeCount >= MAX_CONCURRENT_CONTAINERS) {
    state.pendingMessages = true;
    if (!this.waitingGroups.includes(groupJid)) {
      this.waitingGroups.push(groupJid);  // Fairness: FIFO queue
    }
    return;
  }

  // Run immediately
  this.runForGroup(groupJid, 'messages');
}
```

**3. Waiting Groups FIFO (src/group-queue.ts:318-344)**

```typescript
private drainWaiting(): void {
  while (
    this.waitingGroups.length > 0 &&
    this.activeCount < MAX_CONCURRENT_CONTAINERS
  ) {
    const nextJid = this.waitingGroups.shift()!;  // FIFO
    const state = this.getGroup(nextJid);

    // Tasks take priority over messages
    if (state.pendingTasks.length > 0) {
      this.runTask(nextJid, state.pendingTasks.shift()!);
    } else if (state.pendingMessages) {
      this.runForGroup(nextJid, 'drain');
    }
  }
}
```

**Important: Task Priority Over Messages**
When a group has both pending tasks AND pending messages, tasks run first (line 327). This is because tasks are scheduled/critical work while messages are user-driven.

**4. Active Container Message Piping (src/group-queue.ts:160-178)**

```typescript
sendMessage(groupJid: string, text: string): boolean {
  const state = this.getGroup(groupJid);
  if (!state.active || !state.groupFolder || state.isTaskContainer) {
    return false;  // No active container or it's a task container
  }
  state.idleWaiting = false;

  // Write to IPC input file
  const inputDir = path.join(DATA_DIR, 'ipc', state.groupFolder, 'input');
  fs.mkdirSync(inputDir, { recursive: true });
  const filename = `${Date.now()}-${Math.random().toString(36).slice(2, 6)}.json`;
  fs.writeFileSync(tempPath, JSON.stringify({ type: 'message', text }));
  fs.renameSync(tempPath, filepath);  // Atomic write
  return true;
}
```

**5. Container Lifecycle (src/group-queue.ts:196-231)**

```typescript
private async runForGroup(groupJid: string, reason: 'messages' | 'drain'): Promise<void> {
  const state = this.getGroup(groupJid);
  state.active = true;
  state.idleWaiting = false;
  state.isTaskContainer = false;
  state.pendingMessages = false;
  this.activeCount++;  // Increment BEFORE processing

  try {
    if (this.processMessagesFn) {
      const success = await this.processMessagesFn(groupJid);
      if (!success) {
        this.scheduleRetry(groupJid, state);
      }
    }
  } finally {
    state.active = false;
    this.activeCount--;  // Decrement AFTER completion
    this.drainGroup(groupJid);  // Check for pending work
  }
}
```

**6. Retry with Exponential Backoff (src/group-queue.ts:263-284)**

```typescript
private scheduleRetry(groupJid: string, state: GroupState): void {
  state.retryCount++;
  if (state.retryCount > MAX_RETRIES) {  // 5 max retries
    logger.error({ groupJid }, 'Max retries exceeded, dropping messages');
    state.retryCount = 0;
    return;
  }

  const delayMs = BASE_RETRY_MS * Math.pow(2, state.retryCount - 1);
  // BASE_RETRY_MS = 5000, so: 5s, 10s, 20s, 40s, 80s
  setTimeout(() => {
    if (!this.shuttingDown) {
      this.enqueueMessageCheck(groupJid);
    }
  }, delayMs);
}
```

**7. Idle Notification and Preemption (src/group-queue.ts:148-154)**

```typescript
notifyIdle(groupJid: string): void {
  const state = this.getGroup(groupJid);
  state.idleWaiting = true;
  if (state.pendingTasks.length > 0) {
    this.closeStdin(groupJid);  // Preempt idle container for pending tasks
  }
}
```

When a container finishes and `notifyIdle()` is called, if there are pending tasks, the container is immediately closed to make room for task execution.

**8. Graceful Shutdown (src/group-queue.ts:347-364)**

```typescript
async shutdown(_gracePeriodMs: number): Promise<void> {
  this.shuttingDown = true;

  // Don't kill active containers — they'll finish via idle timeout
  // The --rm flag cleans them up on exit
  const activeContainers: string[] = [];
  for (const [_jid, state] of this.groups) {
    if (state.process && !state.process.killed && state.containerName) {
      activeContainers.push(state.containerName);
    }
  }

  logger.info({ activeCount: this.activeCount, detachedContainers: activeContainers },
    'GroupQueue shutting down (containers detached, not killed)');
}
```

### Clever Solutions

**1. Atomic File Renames for IPC**
Messages are written to a `.tmp` file then renamed (line 172-173), ensuring the container never sees a partial message.

**2. Task Preemption of Idle Containers**
When `notifyIdle()` sees pending tasks, it closes stdin immediately instead of waiting for `IDLE_TIMEOUT`. This ensures scheduled tasks run promptly.

**3. Per-Group Retry State**
Each group tracks its own `retryCount`, so a failing group doesn't affect other groups.

**4. Shutdown Gracefulness**
Containers are "detached" not killed, allowing them to finish naturally. This prevents WhatsApp reconnection restarts from killing working agents.

### Edge Cases

1. **Container crash**: `runForGroup()` catches exceptions, calls `scheduleRetry()`, which will re-enqueue if retries remain
2. **All groups waiting**: `drainWaiting()` loops while `activeCount < MAX_CONCURRENT_CONTAINERS`, so it starts new work as slots free up
3. **Group has both tasks and messages**: Tasks are always processed first (line 327)
4. **Task already running**: `enqueueTask()` checks `runningTaskId` and skips (line 96-98)
5. **Duplicate task enqueue**: `pendingTasks.some(t => t.id === taskId)` prevents double-queuing (line 100-102)

### Configuration

From `src/config.ts:58-61`:
```typescript
export const MAX_CONCURRENT_CONTAINERS = Math.max(
  1,
  parseInt(process.env.MAX_CONCURRENT_CONTAINERS || '5', 10) || 5,
);
```

---

## Cross-Feature Integration

### Message Flow with Queue

```
startMessageLoop()
    |
    v
getNewMessages() --> SQLite
    |
    v
queue.sendMessage() --(if active container)--> IPC file write
    |
    v (if no active container)
queue.enqueueMessageCheck()
    |
    v
GroupQueue.runForGroup()
    |
    v
processGroupMessages() --> runAgent()
    |
    v
Container (Claude Agent SDK)
    |
    v
Streaming callback --> channel.sendMessage()
    |
    v
queue.notifyIdle() --> closeStdin() if pending tasks
```

### Scheduler Integration with Queue

```
startSchedulerLoop()
    |
    v
getDueTasks() --> SQLite
    |
    v
queue.enqueueTask()
    |
    v
GroupQueue.runTask()
    |
    v
task.fn() --> runTask() in task-scheduler.ts
    |
    v
Container (Claude Agent SDK)
    |
    v
sendMessage() --> channel.sendMessage() (result forwarding)
    |
    v
scheduleClose() --> queue.closeStdin()
```

### Shared State

Both features share:
- `GroupQueue` instance (line 75 in `src/index.ts`)
- `registeredGroups` state (line 70 in `src/index.ts`)
- `sessions` state (line 69 in `src/index.ts`)

### IPC File Structure

```
data/ipc/{groupFolder}/
    input/
        {timestamp}-{random}.json  (message pipelining)
        {timestamp}-{random}.json
        _close                     (close sentinel)
    output/
        {timestamp}-{random}.json  (IPC from container)
```

---

## Summary of Key Design Patterns

| Pattern | Implementation | Purpose |
|---------|---------------|---------|
| Polling loops | `while(true)` with `setTimeout` | Non-blocking async operations in Node.js |
| FIFO waiting queue | `waitingGroups.shift()` | Fairness across groups |
| Atomic file writes | `writeFileSync` + `renameSync` | Prevent partial IPC messages |
| Time anchoring | `task.next_run + interval` instead of `now + interval` | Prevent schedule drift |
| Exponential backoff | `BASE_RETRY_MS * Math.pow(2, n)` | Graceful degradation |
| Cursor rollback | Per-group `lastAgentTimestamp` | Exactly-once message processing |
| Task preemption | `notifyIdle()` checks `pendingTasks` | Priority for scheduled work |
| Graceful shutdown | Containers detached, not killed | Prevent work loss on restart |
