# Features Batch 1 Deep Dive (Features 1-3)

## Feature 1: Multi-Agent Orchestration

### Overview
The project implements a sophisticated multi-agent orchestration system with 11 specialized agents. The main orchestrator is **Sisyphus**, which plans, delegates, and drives tasks to completion. The system supports aggressive parallel execution with subagents handling specialized work.

### Core Architecture

#### Agent Type System (`src/agents/types.ts`)
```typescript
export type AgentMode = "primary" | "subagent" | "all";
// - "primary": Respects user's UI-selected model (sisyphus, atlas)
// - "subagent": Uses own fallback chain, ignores UI selection
// - "all": Available in both contexts
```

The `AgentFactory` type combines the agent factory function with a static `mode` property for pre-instantiation access.

#### Builtin Agents (`src/agents/builtin-agents.ts`)
11 agents are registered in `agentSources`:
- **sisyphus** - Main orchestrator
- **hephaestus** - Autonomous deep worker
- **oracle** - Architecture/debugging consultant
- **librarian** - Documentation and code search
- **explore** - Fast codebase exploration
- **multimodal-looker** - Visual analysis
- **metis** - Plan consultant
- **momus** - Plan reviewer
- **atlas** - Todo-list orchestrator
- **sisyphus-junior** - Category-based executor

#### Agent Prompt Metadata (`src/agents/types.ts`)
Each agent has `AgentPromptMetadata` that drives dynamic prompt sections:
```typescript
interface AgentPromptMetadata {
  category: AgentCategory;  // exploration | specialist | advisor | utility
  cost: AgentCost;          // FREE | CHEAP | EXPENSIVE
  triggers: DelegationTrigger[];  // Domain → when to delegate
  useWhen?: string[];
  avoidWhen?: string[];
  dedicatedSection?: string;  // Special sections like Oracle
  promptAlias?: string;       // Display name
  keyTrigger?: string;        // Phase 0 triggers
}
```

### Sisyphus Orchestrator (`src/agents/sisyphus.ts`)

Sisyphus is the central orchestrator. The `createSisyphusAgent()` function builds the agent config with extensive prompt engineering.

**Key Prompt Sections:**
1. **Intent Gate (Phase 0)** - Every message checks intent classification
2. **Phase 1** - Codebase assessment for open-ended tasks
3. **Phase 2A** - Exploration & Research (parallel execution default)
4. **Phase 2B** - Implementation with delegation
5. **Phase 2C** - Failure recovery
6. **Phase 3** - Completion criteria

**Intent Verbalization Map:**
| Surface Form | True Intent | Routing |
|---|---|---|
| "explain X" | Research | explore/librarian |
| "implement X" | Implementation | plan → delegate |
| "look into X" | Investigation | explore |
| "what do you think" | Evaluation | evaluate → wait |
| "error X" | Fix needed | diagnose → fix |
| "refactor" | Open-ended | assess → propose |

**Model-Specific Handling:**
- GPT-5.4: Uses specialized `buildGpt54SisyphusPrompt()`
- Gemini: Injects IntentGate enforcement, Tool mandate, Tool guide, Delegation/verification overrides
- Non-Claude: Adds mandatory Plan Agent consultation
- Claude: Uses `thinking: { type: "enabled", budgetTokens: 32000 }`

### Dynamic Prompt Builder (`src/agents/dynamic-agent-prompt-builder.ts`)

This is the prompt engineering powerhouse. Key functions:

- `buildKeyTriggersSection()` - Phase 0 triggers
- `buildToolSelectionTable()` - Sorted by cost (FREE first)
- `buildDelegationTable()` - Domain → Agent mapping
- `buildExploreSection()` - Delegation trust rules
- `buildLibrarianSection()` - Contextual vs Reference grep
- `buildCategorySkillsDelegationGuide()` - Category + Skills protocol
- `buildParallelDelegationSection()` - Aggressive decomposition for non-Claude
- `buildAntiDuplicationSection()` - Critical: don't re-search after delegating

**Anti-Duplication Rule (CRITICAL):**
Once explore/librarian is fired, do NOT manually perform the same search. This wastes tokens and causes confusion.

### Category System

The `task()` tool supports categories that map to optimized models:
- `visual-engineering` - UI/UX/CSS tasks
- `ultrabrain` - Deep reasoning
- `deep` - Autonomous research + implementation
- `artistry` - Creative tasks
- `quick` - Single-file changes
- `unspecified-high/unspecified-low` - General tasks

### Clever Solutions

1. **Agent Mode Inheritance**: Primary vs subagent modes allow flexible routing
2. **Dynamic Prompt Composition**: Modular sections assembled based on context
3. **Model-Specific Overrides**: Gemini gets extra enforcement sections due to "lost-in-the-middle" attention issues
4. **Phase 0 Intent Gate**: Forces classification before any action
5. **Delegation Trust Rules**: Clear anti-patterns prevent wasted work

### Technical Debt / Concerns

1. **Complex Conditional Prompt Building**: Model-specific handling involves complex string replacement in `createSisyphusAgent()`. Line 493-512 does string replacement to inject Gemini-specific sections. Fragile if prompt structure changes.

2. **Hardcoded Model Detection**: `isGeminiModel()`, `isGptModel()`, `isGpt5_4Model()` use string matching. Adding new models requires code changes.

3. **Sisyphus Junior Bypass**: Cannot use `subagent_type="sisyphus-junior"` directly - must use categories. This is intentional but confusing in error messages.

4. **Plan Family Agent Guard**: Plan agents cannot delegate to other plan agents - creates logical deadlock protection but adds complexity.

---

## Feature 2: Hash-Anchored Edit Tool (Hashline)

### Overview
An edit tool that tags every line with a content hash (`LINE#ID` format). When editing, the hash validates that the file hasn't changed since the last read. This eliminates stale-line errors entirely.

**Impact**: Improved Grok Code Fast success rate from 6.7% to 68.3%.

### Core Algorithm

#### Hash Dictionary (`src/tools/hashline-edit/constants.ts`)
```typescript
const NIBBLE_STR = "ZPMQVRWSNKTXJBYH"
export const HASHLINE_DICT = Array.from({ length: 256 }, (_, i) => {
  const high = i >>> 4
  const low = i & 0x0f
  return `${NIBBLE_STR[high]}${NIBBLE_STR[low]}`
})
// Produces 2-character hash from 0-255 index
// Pattern: /([0-9]+)#([ZPMQVRWSNKTXJBYH]{2})$/
```

#### Hash Computation (`src/tools/hashline-edit/hash-computation.ts`)
```typescript
function computeNormalizedLineHash(lineNumber: number, normalizedContent: string): string {
  const stripped = normalizedContent
  const seed = RE_SIGNIFICANT.test(stripped) ? 0 : lineNumber
  // If line has significant content (letters/numbers): seed=0
  // If line is whitespace-only: seed=lineNumber (ensures different hash)
  const hash = Bun.hash.xxHash32(stripped, seed)
  const index = hash % 256
  return HASHLINE_DICT[index]
}
```

**Clever**: Whitespace-only lines use line number as seed to differentiate them. Non-empty lines use seed=0 so identical content produces same hash across files.

#### Line Format
```
line_number#hash|content
Example: 5#PM|const x = 1;
```

#### Legacy Hash Support
`computeLegacyLineHash()` normalizes by removing ALL whitespace (`replace(/\s+/g, "")`), whereas current hash only trims whitespace (`trimEnd()`). This provides backwards compatibility with older line references.

### Edit Operations (`src/tools/hashline-edit/edit-operation-primitives.ts`)

Supports 5 operations:
1. **replace** (setLine) - Replace single line anchor
2. **replace** (range) - Replace range from startAnchor to endAnchor
3. **append** (anchored) - Insert after anchor line
4. **append** (unanchored) - Append to end of file
5. **prepend** (anchored) - Insert before anchor line
6. **prepend** (unanchored) - Prepend to start of file

**Validation**: Each operation (unless `skipValidation: true`) validates line references before applying.

### Validation (`src/tools/hashline-edit/validation.ts`)

```typescript
export function validateLineRefs(lines: string[], refs: string[]): void {
  // 1. Parse each ref: extract line number and hash
  // 2. Bounds check: line < 1 || line > lines.length
  // 3. Hash match: computeLineHash(line, content) === hash
  // 4. If mismatch: throw HashlineMismatchError with remapping
}
```

**HashlineMismatchError** provides:
- Context lines (±2 lines around mismatch)
- `>>> ` prefix on changed lines
- Automatic remap suggestions for likely typos

### Edit Execution Flow (`src/tools/hashline-edit/edit-operations.ts`)

```typescript
export function applyHashlineEditsWithReport(content: string, edits: HashlineEdit[]) {
  // 1. Deduplicate edits (same pos/lines)
  // 2. Sort: by line DESC, then precedence (replace=0, append=1, prepend=2)
  // 3. Validate all line refs in one pass
  // 4. Detect overlapping ranges (throw if overlap)
  // 5. Apply each edit
  // 6. Return { content, noopEdits, deduplicatedEdits }
}
```

**Processing order**: Edits applied bottom-to-top (highest line first) to preserve line numbers of pending edits.

### Hashline Mismatch Error Recovery

When hash mismatch detected (`HashlineMismatchError`):
- Shows which lines changed with `>>>` marker
- Provides actual vs expected hash
- Suggests corrected line reference if content matches elsewhere
- Error message format:
```
3 lines have changed since last read. Use updated {line_number}#{hash_id} references below (>>> marks changed lines).

>>> 15#XX|const newContent = "updated";
    16#YY|const other = 2;
```

### Clever Solutions

1. **Whitespace Line Differentiation**: Using line number as seed for whitespace-only lines ensures identical whitespace gets different hashes per line.

2. **Legacy Hash Compatibility**: Supporting both normalized (trimEnd) and fully whitespace-removed hashes for backwards compatibility.

3. **Single-Pass Validation**: All line refs validated before any edit applied - prevents partial corruption.

4. **Noop Edit Detection**: Tracks edits that produce identical content (e.g., replacing line with its current content).

5. **Autocorrect Replacement Lines** (`autocorrect-replacement-lines.ts`): Handles indentation when replacing lines - strips boundary echoes and normalizes indentation.

6. **Hash Suggestion**: When hash doesn't match but similar hash exists elsewhere, suggests the correct line reference.

### Edge Cases Handled

- **Overlapping ranges**: Detected and rejected before any edit
- **Empty file**: Special case - `append`/`prepend` create from nothing
- **File doesn't exist**: `canCreateFromMissingFile()` only allows `append`/`prepend` without position
- **CRLF handling**: `content.replace(/\r/g, "")` normalizes before processing
- **Trailing newlines**: Properly handled via `trimEnd()` normalization
- **Delete mode**: `delete=true` with empty edits array

### Hooks Integration

- **hashline-read-enhancer**: Adds hash markers to read output
- **hashline-edit-diff-enhancer**: Enhances edit output with diff markers

---

## Feature 3: Background Agents & Parallel Execution

### Overview
Fire multiple specialized agents in parallel, continuing main work while specialists produce results. Supports tmux integration for visual multi-agent sessions.

### Two Execution Models

#### 1. Background Task (`run_in_background=true`)
- Returns immediately with `task_id`
- System notifies via `<system-reminder>` when complete
- Fire-and-forget with result collection via `background_output(task_id)`
- Used for parallel exploration, librarian research

#### 2. Sync Task (`run_in_background=false`)
- Blocks until task completes
- Returns full result text
- Used for task delegation that needs immediate response
- Has timeout handling via `syncPollTimeoutMs`

### Background Task Architecture

#### BackgroundManager (`src/features/background-agent/manager.ts`)

Core class managing all background tasks. Key features:

**Task Lifecycle:**
```
pending → running → completed/error/cancelled/interrupt
```

**Launch Flow:**
```typescript
async launch(input: LaunchInput): Promise<BackgroundTask> {
  // 1. Reserve subagent spawn (checks depth/budget limits)
  // 2. Create task with status="pending"
  // 3. Add to queue by concurrency key
  // 4. Process queue (fire-and-forget)
  // 5. Return immediately
}
```

**Key Data Structures:**
- `tasks: Map<string, BackgroundTask>` - All tasks
- `queuesByKey: Map<string, QueueItem[]>` - Pending tasks by concurrency key
- `notifications: Map<string, BackgroundTask[]>` - Pending notifications per parent
- `rootDescendantCounts: Map<string, number>` - Tracks subagent tree depth

**Concurrency Control:**
```typescript
private concurrencyManager: ConcurrencyManager
// Limits concurrent tasks per model/agent
// Prevents resource exhaustion
```

**Session Management:**
- Creates child session via `client.session.create()`
- Inherits parent directory
- Sets `parentID` for session hierarchy
- Supports `sessionPermission` inheritance

**Polling:**
- `POLLING_INTERVAL_MS` - How often to check task status
- `TASK_TTL_MS` - Task time-to-live before cleanup
- `TASK_CLEANUP_DELAY_MS` - Delay before removing completed tasks

### Task Delegation (`src/tools/delegate-task/`)

#### DelegateTask Tool (`tools.ts`)

Main interface. **Critical requirement**: Must provide EITHER `category` OR `subagent_type`.

```typescript
interface DelegateTaskArgs {
  load_skills: string[];        // REQUIRED - skills to inject
  description: string;          // Short description
  prompt: string;               // Full prompt
  run_in_background: boolean;   // REQUIRED
  category?: string;             // Category-based OR
  subagent_type?: string;       // Direct agent
  session_id?: string;          // Continue existing session
  command?: string;             // Slash command tracking
}
```

**Session Continuation:**
- If `session_id` provided, continues existing agent session
- Preserves full context, saves tokens
- Critical for multi-turn with same agent

#### Category Resolution (`category-resolver.ts`)

Resolves category to agent model configuration:
```typescript
resolveCategoryExecution(args, executorCtx, inheritedModel, systemDefaultModel)
// Returns: { agentToUse, categoryModel, fallbackChain, isUnstableAgent, error? }
```

**Model Resolution Precedence:**
1. Explicit category model (user configured)
2. `sisyphus-junior.model` override
3. Category resolved model (system default)

**Unstable Agent Detection:**
```typescript
const isUnstableAgent = resolvedModel?.includes("gemini") ||
                        resolvedModel?.includes("minimax") ||
                        resolvedModel?.includes("kimi");
// Unstable agents: auto-forced to background execution
```

**Fallback Chains:**
```typescript
fallbackChain = configuredFallbackChain ?? requirement?.fallbackChain
// Supports runtime retry on model errors (429, 503, 529)
```

#### Subagent Resolution (`subagent-resolver.ts`)

For direct agent invocation (explore, librarian, oracle, etc.):
- Validates agent exists and is callable (`mode !== "primary"`)
- Cannot call primary agents (sisyphus, atlas) via task
- Plan family agents cannot delegate to other plan agents
- Resolves model with fallback chain support

### Sync Execution (`sync-task.ts`)

Blocking execution with polling:
```typescript
async executeSyncTask(args, ctx, executorCtx, parentContext, ...) {
  // 1. Reserve subagent spawn slot
  // 2. Create sync session
  // 3. Register for notifications
  // 4. Send prompt
  // 5. Poll for completion (with timeout)
  // 6. Fetch result
  // 7. Return formatted output
}
```

**Timeout Handling:**
- `syncPollTimeoutMs` - Configurable timeout
- Aborts session on timeout
- Cleans up resources

**Result Format:**
```
Task completed in 45s.

Agent: sisyphus-junior (category: deep)

---

[Result text content]

<task_metadata>
session_id: ses_abc123
</task_metadata>
```

### Background Task Execution (`background-task.ts`)

Fire-and-forget:
```typescript
async executeBackgroundTask(...) {
  // 1. Build prompt
  // 2. Launch via BackgroundManager
  // 3. Wait briefly for session creation (50ms intervals, 30s timeout)
  // 4. Register session for fallback chain
  // 5. Return task info immediately
}
```

### Notification System

**Parent Session Notification:**
When task completes, `notifyParentSession()`:
1. Shows toast notification
2. Injects `<system-reminder>` into parent's next turn
3. Batches notifications for multiple tasks

**Notification Format (all complete):**
```xml
<system-reminder>
[ALL BACKGROUND TASKS COMPLETE]
Completed:
- `bg_abc123`: Find auth patterns
- `bg_def456`: Find error handling

Use `background_output(task_id="<id>")` to retrieve each result.
</system-reminder>
```

**Notification Format (individual):**
```xml
<system-reminder>
[BACKGROUND TASK COMPLETED]
ID: `bg_abc123`
Description: Find auth patterns
Duration: 45s

2 tasks still in progress. You WILL be notified when ALL complete.
Do NOT poll - continue productive work.
</system-reminder>
```

### Circuit Breaker / Loop Detection

**Tool Call Counting:**
```typescript
task.progress.toolCalls++
if (toolCalls >= maxToolCalls) {
  cancelTask(..., "circuit-breaker")
}
```

**Repetitive Tool Detection:**
```typescript
detectRepetitiveToolUse(toolCallWindow)
// Tracks consecutive tool calls
// Cancels if threshold exceeded
```

**Stale Task Detection:**
```typescript
checkAndInterruptStaleTasks()
// Tasks with no progress for extended time
// Session idle but incomplete todos
```

### Spawn Limits

**Depth Limits:**
```typescript
const maxDepth = getMaxSubagentDepth(config)
if (childDepth > maxDepth) {
  throw createSubagentDepthLimitError(...)
}
```

**Root Session Budget:**
```typescript
const maxDescendants = getMaxRootSessionSpawnBudget(config)
if (descendantCount >= maxDescendants) {
  throw createSubagentDescendantLimitError(...)
}
```

### Unstable Agent Handling

When an unstable model (Gemini/MiniMax/Kimi) is detected with `run_in_background=false`:
```typescript
if (isUnstableAgent && isRunInBackgroundExplicitlyFalse) {
  return executeUnstableAgentTask(...) // Auto-forces background
}
```

This prevents blocking on unreliable models.

### Session Tracking

**Subagent Sessions Registry:**
```typescript
subagentSessions: Set<string>  // All active subagent sessions
syncSubagentSessions: Set<string>  // Sync-only sessions
```

**Category Registry:**
```typescript
SessionCategoryRegistry.register(sessionId, category)
// Used for analytics and tool display
```

### Tmux Integration

When tmux enabled and inside tmux session:
- `onSubagentSessionCreated` callback invoked
- Creates visual tmux pane for each subagent
- Real-time output streaming

### Error Handling

**Fallback Retry:**
```typescript
tryFallbackRetry(task, errorInfo, source)
// On 429/503/529: retry with next model in chain
// Increments attemptCount
// Re-launches with different model
```

**Graceful Degradation:**
- Task errors don't crash parent session
- Error messages embedded in notification
- Session aborted on terminal errors

### Clever Solutions

1. **Queue-based Concurrency**: Tasks queued per model/agent key, processed based on concurrency limits.

2. **Pre-start Reservation**: `reserveSubagentSpawn()` checks limits BEFORE creating task, prevents zombie tasks.

3. **Atomic Task State Transitions**: `tryCompleteTask()` uses atomic status check to prevent double-completion.

4. **Notification Batching**: Queues notifications per parent, sends single combined notification when all complete.

5. **Idle Deferral**: Tasks briefly idle are given grace period before marking complete, prevents premature completion.

6. **Compaction-Aware Message Resolution**: During session compaction, falls back to file-based message storage.

7. **Fallback Chain Persistence**: `setSessionFallbackChain()` persists retry chain for runtime errors.

### Technical Debt / Concerns

1. **Complex Polling Logic**: `pollRunningTasks()` has extensive conditional logic for session status handling (active, terminal, idle, unknown).

2. **Timer Management**: Multiple timer maps (`completionTimers`, `idleDeferralTimers`) with cleanup logic scattered across methods.

3. **Race Condition Protection**: `pollingInFlight` flag prevents concurrent polling runs - necessary but adds complexity.

4. **Spawn Reservation Rollback**: Two-phase commit pattern for spawn limits - `commit()` and `rollback()` must be carefully matched.

5. **Circular Dependency in Cleanup**: Task completion → notification → cleanup → session removal has many edge cases handled via `scheduleTaskRemoval()`.

6. **Cold Cache Model Resolution**: When model cache is empty, resolution is "skipped" but explicit user overrides are still honored. Complex conditional logic in `subagent-resolver.ts`.

---

## Cross-Feature Interactions

### Hashline + Background Agents
- Edit operations run in main session
- Background agents may read files with hashline markers
- No direct coupling - hashline is transparent to callers

### Multi-Agent Orchestration + Background Agents
- Sisyphus delegates to subagents via `task()` tool
- `task(run_in_background=true)` spawns background agents
- Session hierarchy: parent creates child sessions
- Sisyphus prompt explicitly instructs parallel delegation

### Multi-Agent Orchestration + Hashline
- Sisyphus uses `edit` tool extensively
- Hashline validation prevents stale edits
- Error recovery includes re-reading and re-editing

---

## Summary Table

| Aspect | Feature 1 | Feature 2 | Feature 3 |
|--------|-----------|-----------|-----------|
| **Lines of Code** | ~2000 (agents/) | ~3000 (hashline-edit/) | ~4000 (background-agent/, delegate-task/) |
| **Test Coverage** | Some unit tests | Extensive tests | Extensive tests |
| **Complexity** | High (prompt engineering) | Medium (algorithm) | Very High (concurrency) |
| **Key Pattern** | Factory + Strategy | Hash table + validation | Queue + event + polling |
| **Failure Modes** | Model routing issues | Hash collision edge cases | Race conditions, zombie sessions |
| **Extensibility** | Add new agents easily | Add new operations | Add new executors |
