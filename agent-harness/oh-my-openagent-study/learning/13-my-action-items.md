# 13 My Action Items: Building a Similar Project

## Context

Based on research into oh-my-openagent (oh-my-opencode v3.11.0), these are prioritized action items for building a similar AI agent harness plugin system.

---

## Priority 1: Foundation (Do First)

### 1.1 Define Hook Interface and Event System

**Why:** Hook-based architecture is the core abstraction. Everything else builds on it.

**Action:** Create `src/plugin/hooks.ts` with:
```typescript
type HookName = 'session.created' | 'session.idle' | 'session.error' |
  'tool.execute.before' | 'tool.execute.after' | 'chat.message' |
  'experimental.chat.messages.transform' | 'experimental.chat.system.transform'

type Hook = {
  [K in HookName]?: (input: any, output?: any) => Promise<void>
  dispose?: () => void
}

type HookFactory = (ctx: PluginContext) => Hook
```

**Specifics:**
- Central event dispatcher (`dispatchToHooks()`)
- Hook registration with enable/disable via config
- Error isolation per hook (one hook failure shouldn't crash system)

**Verification:** Write test that creates two hooks, dispatches event, verifies both called.

---

### 1.2 Set Up Bun Workspace with Strict TypeScript

**Why:** oh-my-openagent uses Bun as unified toolchain. Start with this to avoid later migration.

**Action:**
```bash
bun init
bun add -d typescript bun-types
```

**Action:** Create `tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "declaration": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "lib": ["ESNext"],
    "types": ["bun-types"]
  },
  "exclude": ["**/*.test.ts"]
}
```

**Action:** Add linting immediately:
```bash
bun add -d eslint prettier @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

**Create `.eslintrc.json`:**
```json
{
  "parser": "@typescript-eslint/parser",
  "extends": ["plugin:@typescript-eslint/recommended"],
  "rules": {
    "@typescript-eslint/no-explicit-any": "warn",
    "@typescript-eslint/explicit-function-return-type": "off"
  }
}
```

**Verification:** `bun run lint` and `bun run typecheck` pass in CI.

---

### 1.3 Build Zod Config Schema System

**Why:** Runtime validation catches config errors early with clear messages.

**Action:** Create `src/config/schema.ts`:
```typescript
import { z } from 'zod'

export const AgentConfigSchema = z.object({
  model: z.string().optional(),
  prompt_append: z.string().optional(),
  tools: z.array(z.string()).optional(),
})

export const MyPluginConfigSchema = z.object({
  $schema: z.string().optional(),
  default_agent: z.string().default('sisyphus'),
  agents: z.record(z.string(), AgentConfigSchema).optional(),
  disabled_hooks: z.array(z.string()).optional(),
})
```

**Pattern to follow:** Each config section gets its own schema file in `src/config/schema/`. Main schema imports and composes them.

**Verification:** Write tests for valid/invalid config cases.

---

## Priority 2: Core Agent System (Do Second)

### 2.1 Implement Agent Factory Pattern

**Why:** Agents are parameterized factories that receive model name and return config.

**Action:** Create `src/agents/types.ts`:
```typescript
export type AgentMode = 'primary' | 'subagent' | 'all'

export interface AgentFactory {
  (model: string): AgentConfig
  mode: AgentMode
}

export interface AgentConfig {
  systemPrompt: string
  tools?: string[]
  thinking?: { type: 'enabled' | 'disabled'; budgetTokens?: number }
}
```

**Action:** Create `src/agents/index.ts` with builtin agent registrations.

**Follow oh-my-openagent's pattern:** Each agent gets:
- Base implementation with dynamic prompt sections
- Model-specific prompt variants (Claude vs GPT vs Gemini)
- `mode` property for routing (primary = respects UI selection, subagent = uses own chain)

**Verification:** Write tests for agent resolution given different model names.

---

### 2.2 Build Category-Based Delegation

**Why:** Users shouldn't pick models; they should pick task types.

**Action:** Create `src/shared/categories.ts`:
```typescript
export const DEFAULT_CATEGORIES = {
  'visual-engineering': { model: 'claude-sonnet-4', prompt_append: 'UI/UX task...' },
  'ultrabrain': { model: 'claude-opus-4', prompt_append: 'Deep reasoning...' },
  'deep': { model: 'claude-sonnet-4', prompt_append: 'Autonomous research...' },
  'quick': { model: 'claude-haiku-4', prompt_append: 'Single-file changes...' },
  'writing': { model: 'claude-sonnet-4', prompt_append: 'Anti-AI-slop rules...' },
} as const

export type CategoryName = keyof typeof DEFAULT_CATEGORIES
```

**Action:** Create `src/tools/delegate-task.ts` tool that accepts `category` parameter and resolves to appropriate agent.

**Follow their pattern:** Category resolution should handle:
- Explicit category model > category default > system default
- Fallback chains for retry
- Unstable agent detection (Gemini/MiniMax/Kimi auto-forced to background)

**Verification:** Write tests for each category resolving to expected defaults.

---

### 2.3 Implement Background Task Manager

**Why:** Parallel agent execution requires safety systems.

**Action:** Create `src/managers/background-task-manager.ts` with:
```typescript
export class BackgroundTaskManager {
  async launch(input: LaunchInput): Promise<BackgroundTask>
  async pollRunningTasks(): Promise<void>
  async cancel(taskId: string): Promise<void>
}
```

**Implement these safety systems:**
1. **Circuit breaker:** Track `toolCalls` per task, cancel at `maxToolCalls`
2. **Repetitive tool detection:** Cancel if same tool called N times consecutively
3. **Depth limits:** Prevent subagent recursion beyond `maxDepth`
4. **Spawn budget:** Limit total descendants from root session
5. **Idle deferral:** Give tasks grace period before marking complete

**Follow their pattern:** State machine for tasks: `pending -> running -> completed/error/cancelled`

**Verification:** Write tests for each safety system triggering correctly.

---

## Priority 3: Essential Tools (Do Third)

### 3.1 Build Hashline Edit Tool

**Why:** Solves stale edit problem. Concrete 10x improvement (6.7% to 68.3% success).

**Action:** Create `src/tools/hashline-edit/` with:
- `constants.ts`: Hash dictionary (256 2-char codes from `ZPMQVRWSNKTXJBYH`)
- `hash-computation.ts`: `computeLineHash()` using `Bun.hash.xxHash32()`
- `validation.ts`: `validateLineRefs()` checking bounds and hash match
- `edit-operations.ts`: `applyHashlineEditsWithReport()` with overlap detection
- `error.ts`: `HashlineMismatchError` with remapping suggestions

**Key algorithm:**
```typescript
const seed = isWhitespaceOnly ? lineNumber : 0
const hash = Bun.hash.xxHash32(strippedContent, seed)
const index = hash % 256
return HASHLINE_DICT[index]
```

**Follow their pattern:** Single-pass validation, bottom-to-top edit application, noop detection.

**Verification:** Write tests for hash computation, validation success/failure, edge cases (CRLF, empty file, etc.).

---

### 3.2 Build LSP Integration

**Why:** IDE-precision tools differentiate the agent harness from basic chatbots.

**Action:** Create `src/tools/lsp/` with:
- `lsp-client.ts`: Document sync, request dispatch, response parsing
- `lsp-process.ts`: Server spawning with Windows workaround (use Node.js spawn on Win32)
- `server-definitions.ts`: 30+ builtin servers (typescript, python, rust, go, etc.)
- `server-resolution.ts`: Priority chain (project > user > opencode > builtin)

**Implement these tools:**
- `lsp_goto_definition`: Jump to symbol
- `lsp_find_references`: Find all usages
- `lsp_rename`: Safe rename with preparation step
- `lsp_diagnostics`: Errors/warnings from language server

**Follow their pattern:** 1-second sleep after initial open for server initialization, graceful degradation on diagnostic failure.

**Verification:** Write tests for server resolution, document sync, response parsing.

---

### 3.3 Build Task System with Dependency Management

**Why:** File-based tasks with `blockedBy`/`blocks` relationships enable wave-based parallel execution.

**Action:** Create `src/tools/task/` with:
- `task-create.ts`: Generate `T-{uuid}` IDs, atomic JSON writes
- `task-list.ts`: Filter blockedBy to unresolved blockers only
- `task-update.ts`: `addBlockedBy`/`addBlocks` for dependency management
- `storage.ts`: Atomic writes via temp file rename, lock acquisition

**Implement continuation enforcer:**
```typescript
// src/hooks/todo-continuation-enforcer.ts
// On session.idle with incomplete todos:
// 1. Check stagnation count
// 2. If progress made, reset stagnation
// 3. If stagnation >= 3, stop continuation
// 4. Otherwise, inject prompt to continue
```

**Follow their pattern:** 14 distinct skip conditions before continuation (background tasks running, pending questions, cooldown period, etc.)

**Verification:** Write tests for dependency filtering, atomic writes, stagnation detection.

---

## Priority 4: Context Management (Do Fourth)

### 4.1 Implement Context Window Monitor

**Why:** Proactive monitoring prevents hitting limits unexpectedly.

**Action:** Create `src/hooks/context-window-monitor.ts`:
```typescript
const CONTEXT_WARNING_THRESHOLD = 0.70 // 70%
const PREEMPTIVE_COMPACTION_THRESHOLD = 0.78 // 78%

// Track via message.updated events
// On tool.execute.after: check usage ratio
// If > 70%, show warning
// If > 78%, trigger preemptive compaction
```

**Action:** Create `src/shared/dynamic-truncator.ts`:
```typescript
// Gets actual context window usage
// Calculates max output tokens as min(remaining * 0.5, targetMaxTokens)
// Truncates with [X more lines truncated] notice
```

**Follow their pattern:** Token estimation via `chars / 4` (rough but fast), header preservation (first 3 lines).

**Verification:** Write tests for truncation at various context levels.

---

### 4.2 Implement Session Recovery Hooks

**Why:** AI agents make mistakes; system should recover automatically.

**Action:** Create `src/hooks/session-recovery.ts`:
```typescript
// Detect error types:
// - tool_result_missing: Tool used without result
// - thinking_block_order: Blocks in wrong order
// - thinking_disabled_violation: Block used when disabled

// Recovery mechanisms:
// - Inject cancelled tool results
// - Strip or reorder thinking blocks
// - Auto-resume on recovery
```

**Action:** Create `src/hooks/runtime-fallback.ts`:
```typescript
// On session.error:
// - Check if retryable (429, 503, 529, rate limit errors)
// - Extract fallback models
// - Schedule retry with next model in chain
```

**Follow their pattern:** Error classification with RETRYABLE vs NON_RETRYABLE sets, session state tracking with TTL-based cleanup.

**Verification:** Write tests for error type detection and recovery injection.

---

## Priority 5: Quality Hooks (Do Fifth)

### 5.1 Build Write Existing File Guard

**Why:** Prevents agents from overwriting files without reading them first.

**Action:** Create `src/hooks/write-existing-file-guard.ts`:
```typescript
// Permission model:
// - read tool grants write permission (session-scoped, path-canonicalized)
// - write tool consumes permission
// - overwrite: true flag bypasses guard

// LRU eviction at MAX_TRACKED_SESSIONS (256)
// Path canonicalization via realpathSync.native
```

**Follow their pattern:** 18 test scenarios covering all edge cases (symlinks, relative/absolute paths, cross-session, etc.)

**Verification:** Write tests for each scenario.

---

### 5.2 Build Edit Error Recovery

**Why:** AI agents make specific edit mistakes; guide them to fix.

**Action:** Create `src/hooks/edit-error-recovery.ts`:
```typescript
const EDIT_ERROR_PATTERNS = [
  "oldString and newString must be different",
  "oldString not found",
  "oldString found multiple times",
]

const EDIT_ERROR_REMINDER = `
[EDIT ERROR - IMMEDIATE ACTION REQUIRED]
1. READ the file immediately
2. VERIFY actual content
3. APOLOGIZE briefly
4. CONTINUE with corrected action
`
```

**On tool.execute.after for "edit" tool:**
- Check output for error patterns (case-insensitive)
- Append recovery reminder

**Verification:** Write tests for each error pattern detection.

---

### 5.3 Build Comment Checker

**Why:** AI agents add excessive comments; enforce style automatically.

**Note:** oh-my-openagent uses an external Go binary. For simplicity, use a native TypeScript approach or a lightweight external tool.

**Action:** Create `src/hooks/comment-checker.ts`:
```typescript
// On tool.execute.after for write/edit:
// - Skip if output indicates failure
// - Run comment analysis (external CLI or TypeScript analyzer)
// - If excessive, append warning

// Smart ignoring:
// - BDD tests (given/when/then)
// - Directives (eslint-disable)
// - Docstrings
```

**Verification:** Write tests for comment detection and smart ignoring.

---

## Priority 6: Polish (Do Last)

### 6.1 Add Session Notifications

**Why:** Users need to know when background agents complete.

**Action:** Create `src/hooks/session-notification.ts`:
```typescript
// Platform detection (darwin/linux/win32)
// Idle notification scheduling (1.5s delay)
// Activity grace period to prevent race conditions
// Notification types: idle, question, permission
```

**Follow their pattern:** `terminal-notifier` on macOS, `notify-send` on Linux, PowerShell toast on Windows.

**Verification:** Manual test on each platform.

---

### 6.2 Add Auto-Update Checker

**Why:** Users should know when new versions are available.

**Action:** Create `src/hooks/auto-update-checker.ts`:
```typescript
// On session.created (non-child session):
// - Get cached version
// - Check npm for newer version
// - Show toast if update available
// - Skip local dev mode
```

**Verification:** Mock npm registry response, verify toast shown.

---

## Testing Strategy

### Test File Location
Follow oh-my-openagent's pattern: co-located `.test.ts` files in same directories as source.

### Test Isolation
```typescript
// bunfig.toml
[test]
preload = "./test-setup.ts"

// test-setup.ts
beforeEach(() => {
  resetState()
  clearCaches()
})
```

### Mock-Heavy Test Isolation
For tests using `mock.module()`, run in separate processes:
```yaml
# CI: split tests into isolated groups
- bun test src/plugin-handlers  # mock-heavy
- bun test src/hooks/atlas       # mock-heavy
- bun test src/config            # no mocks
```

---

## Non-Goals (Deliberately Excluded)

Based on research, these are NOT priorities for initial implementation:

1. **Claude Code compatibility layer** - Add later, after core is stable
2. **Tmux integration** - Feature creep; shell is sufficient initially
3. **Multiple MCP servers** - Start with one, add as needed
4. **Prometheus strategic planner** - Complex prompt engineering; defer
5. **Skill-embedded MCPs** - Advanced pattern; defer

---

## Summary Checklist

- [ ] Hook interface and event system
- [ ] Bun workspace with strict TypeScript
- [ ] ESLint + Prettier configured
- [ ] Zod config schema system
- [ ] Agent factory pattern with 3-5 agents
- [ ] Category-based delegation (5 categories)
- [ ] Background task manager with safety systems
- [ ] Hashline edit tool
- [ ] LSP integration (3-5 tools)
- [ ] Task system with dependencies
- [ ] Context window monitor
- [ ] Session recovery hooks
- [ ] Runtime fallback hooks
- [ ] Write existing file guard
- [ ] Edit error recovery
- [ ] Comment checker
- [ ] Session notifications
- [ ] Auto-update checker

**Estimated implementation:** 8-12 weeks for a solo developer, following the priorities above.
