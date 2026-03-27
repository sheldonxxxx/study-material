# oh-my-openagent Features Deep Dive

**Source:** Research synthesis from 6 feature batch files
**Date:** 2026-03-26
**Ordering:** Priority tiers from `04-features-index.md` (Core features first)

---

## Project Overview

oh-my-openagent (OmO) is a plugin for OpenCode that provides multi-agent orchestration, world-class developer tools, and opinionated productivity features. It orchestrates multiple AI models (Claude, Kimi, GLM, GPT, Gemini, Grok) across specialized agents. The project is written in TypeScript with source rooted at `src/`, configuration at `.opencode/` (project-level) and `~/.config/opencode/` (user-level), and platform-specific binaries in `packages/`.

**Notable cross-cutting patterns:**
- Heavy use of hooks (event-driven lifecycle management)
- File-based state persistence for session recovery and loop state
- Model-specific prompt variants throughout
- Concurrency management via queues, semaphores, and atomic operations

---

## Tier 1: Core Features

---

### Feature 1: Multi-Agent Orchestration (11 Discipline Agents)

**Location:** `src/agents/`

The crown jewel of oh-my-openagent. Eleven specialized agents, each tuned to specific model strengths, coordinated by a main orchestrator.

#### Agent Roster

| Agent | Role | File |
|-------|------|------|
| **Sisyphus** | Main orchestrator, plans/delegates/drives | `src/agents/sisyphus.ts` |
| **Hephaestus** | Autonomous deep worker | `src/agents/hephaestus.ts` |
| **Prometheus** | Strategic planner (interview-mode) | `src/agents/prometheus/` |
| **Oracle** | Architecture/debugging consultant | `src/agents/oracle.ts` |
| **Librarian** | Docs/code search | `src/agents/librarian.ts` |
| **Explore** | Fast codebase exploration | `src/agents/explore.ts` |
| **Multimodal-Looker** | Visual analysis | (in `builtin-agents.ts`) |
| **Atlas** | Todo-list orchestrator | `src/agents/atlas/` |
| **Momus** | Plan reviewer | (in `builtin-agents.ts`) |
| **Metis** | Plan consultant (pre-planning) | `src/agents/metis.ts` |
| **Sisyphus-Junior** | Category-based executor | `src/agents/sisyphus-junior/` |

#### Agent Mode System

Three modes control availability and behavior:
- **primary**: Respects the user's UI-selected model (Sisyphus, Atlas)
- **subagent**: Uses its own fallback chain, ignores UI selection
- **all**: Available in both contexts

```typescript
export type AgentMode = "primary" | "subagent" | "all";
```

#### Agent Factory Pattern

Each agent is registered in `agentSources` within `builtin-agents.ts`. The `AgentFactory` type combines the factory function with a static `mode` property for pre-instantiation access to mode information.

#### Dynamic Prompt Builder (`src/agents/dynamic-agent-prompt-builder.ts`)

This is the prompt engineering powerhouse, assembling modular sections based on context:

- `buildKeyTriggersSection()` -- Phase 0 triggers
- `buildToolSelectionTable()` -- Sorted by cost (FREE first)
- `buildDelegationTable()` -- Domain to agent mapping
- `buildExploreSection()` -- Delegation trust rules
- `buildLibrarianSection()` -- Contextual vs reference grep
- `buildCategorySkillsDelegationGuide()` -- Category + skills protocol
- `buildParallelDelegationSection()` -- Aggressive decomposition for non-Claude
- `buildAntiDuplicationSection()` -- Critical: do not re-search after delegating

**Anti-Duplication Rule:** Once explore or librarian is fired, the orchestrator must NOT manually perform the same search. This is a hard rule to prevent wasted tokens and confusion.

#### Sisyphus Orchestration Phases

1. **Phase 0 -- Intent Gate**: Every message is classified before action
2. **Phase 1 -- Codebase Assessment**: For open-ended tasks
3. **Phase 2A -- Exploration & Research**: Parallel execution is the default
4. **Phase 2B -- Implementation**: With delegation
5. **Phase 2C -- Failure Recovery**: When things go wrong
6. **Phase 3 -- Completion Criteria**: Final verification

#### Intent Verbalization Map

A key design that routes surface-level user language to the correct underlying intent:

| Surface Form | True Intent | Routing |
|---|---|---|
| "explain X" | Research | explore/librarian |
| "implement X" | Implementation | plan then delegate |
| "look into X" | Investigation | explore |
| "what do you think" | Evaluation | evaluate then wait |
| "error X" | Fix needed | diagnose then fix |
| "refactor" | Open-ended | assess then propose |

#### Model-Specific Handling

The system applies model-specific prompt overrides:
- **GPT-5.4**: Uses specialized `buildGpt54SisyphusPrompt()`
- **Gemini**: Injects IntentGate enforcement, tool mandate, tool guide, and delegation/verification overrides (due to "lost-in-the-middle" attention issues)
- **Non-Claude**: Adds mandatory Plan Agent consultation
- **Claude**: Uses `thinking: { type: "enabled", budgetTokens: 32000 }`

#### Key Implementation Details

- **Phase 0 Intent Gate**: Forces classification before any action on every message
- **Dynamic Prompt Composition**: Modular sections assembled based on context
- **Model-Specific Overrides**: Gemini-specific sections injected via string replacement (lines 493-512 of `sisyphus.ts`) -- fragile if prompt structure changes
- **Hardcoded Model Detection**: `isGeminiModel()`, `isGptModel()`, `isGpt5_4Model()` use string matching. Adding new models requires code changes.

#### Technical Debt

1. Complex conditional prompt building with string replacement in `createSisyphusAgent()` -- fragile
2. Hardcoded model string matching throughout
3. Sisyphus Junior cannot be used via `subagent_type="sisyphus-junior"` directly -- must go through categories
4. Plan family agents cannot delegate to other plan agents -- intentional deadlock protection that adds complexity

---

### Feature 2: Hash-Anchored Edit Tool (Hashline)

**Location:** `src/tools/hashline-edit/` (38 files)

This is the feature with the most dramatic impact claim in the entire project: it improved Grok Code Fast's success rate from 6.7% to 68.3%. The core idea is simple but powerful: tag every line with a content hash, then validate that hash before any edit to ensure the file has not changed since the last read.

#### Line Format

```
line_number#hash|content
Example: 5#PM|const x = 1;
```

Hash format: 2 characters from the dictionary `ZPMQVRWSNKTXJBYH`, computed via `/([0-9]+)#([ZPMQVRWSNKTXJBYH]{2})$/`.

#### Hash Dictionary

```typescript
const NIBBLE_STR = "ZPMQVRWSNKTXJBYH"
export const HASHLINE_DICT = Array.from({ length: 256 }, (_, i) => {
  const high = i >>> 4
  const low = i & 0x0f
  return `${NIBBLE_STR[high]}${NIBBLE_STR[low]}`
})
```

#### Hash Computation

The clever part is the seed selection:

```typescript
function computeNormalizedLineHash(lineNumber: number, normalizedContent: string): string {
  const stripped = normalizedContent
  const seed = RE_SIGNIFICANT.test(stripped) ? 0 : lineNumber
  // If line has significant content: seed=0 (identical content = same hash across files)
  // If line is whitespace-only: seed=lineNumber (each whitespace line gets unique hash)
  const hash = Bun.hash.xxHash32(stripped, seed)
  const index = hash % 256
  return HASHLINE_DICT[index]
}
```

**Whitespace line differentiation** is the key insight: identical whitespace-only lines (e.g., three blank lines in a row) would produce identical hashes without line-number seeding, making them indistinguishable for editing. By using the line number as seed, each whitespace line gets a unique hash.

**Non-empty lines** use seed=0, so identical content in different locations produces the same hash -- meaning the same content always maps to the same hash, which is correct.

#### Legacy Hash Support

The system maintains backward compatibility with older line references via `computeLegacyLineHash()`, which normalizes by removing ALL whitespace (`replace(/\s+/g, "")`), whereas the current hash only trims via `trimEnd()`. This allows old line references to still resolve correctly.

#### Edit Operations

Six operations supported:
1. **replace (setLine)** -- Replace single line by anchor
2. **replace (range)** -- Replace range from startAnchor to endAnchor
3. **append (anchored)** -- Insert after anchor line
4. **append (unanchored)** -- Append to end of file
5. **prepend (anchored)** -- Insert before anchor line
6. **prepend (unanchored)** -- Prepend to start of file

#### Validation

All line references are validated in a single pass before any edit is applied. The validator:
1. Parses each ref: extracts line number and hash
2. Bounds checks: `line < 1 || line > lines.length`
3. Hash matches: `computeLineHash(line, content) === hash`
4. On mismatch: throws `HashlineMismatchError` with remapping suggestions

#### Edit Execution

`applyHashlineEditsWithReport()`:
1. Deduplicates edits (same position and lines)
2. Sorts by line descending, then by precedence (replace=0, append=1, prepend=2)
3. Validates all line refs in one pass
4. Detects overlapping ranges and throws if overlap found
5. Applies each edit bottom-to-top (highest line first) to preserve line numbers of pending edits
6. Returns `{ content, noopEdits, deduplicatedEdits }`

#### Hashline Mismatch Error Recovery

When a mismatch is detected, the error shows:
- Context lines (2 before and after the mismatch)
- `>>>` prefix on changed lines
- Actual vs expected hash
- Automatic remap suggestions if content matches elsewhere

```
3 lines have changed since last read. Use updated {line_number}#{hash_id} references below (>>> marks changed lines).

>>> 15#XX|const newContent = "updated";
    16#YY|const other = 2;
```

#### Clever Solutions

1. **Whitespace line differentiation via line-number seeding**: Prevents hash collisions on identical blank lines
2. **Legacy hash compatibility**: Two normalization strategies coexist
3. **Single-pass validation**: All refs validated before any edit applied -- prevents partial corruption
4. **Noop edit detection**: Tracks edits producing identical content
5. **Autocorrect replacement lines**: Strips boundary echoes and normalizes indentation when replacing
6. **Hash suggestion**: When hash does not match but similar hash exists elsewhere, suggests the correct line reference

#### Edge Cases Handled

- Overlapping ranges: detected and rejected before any edit
- Empty file: special case for append/prepend
- File doesn't exist: `canCreateFromMissingFile()` only allows append/prepend without position
- CRLF handling: `content.replace(/\r/g, "")` normalizes before processing
- Trailing newlines: properly handled via `trimEnd()` normalization
- Delete mode: `delete=true` with empty edits array

---

### Feature 3: Background Agents & Parallel Execution

**Location:** `src/tools/background-task/` (25 files), `src/tools/delegate-task/` (56 files)

Two execution models:

1. **Background Task** (`run_in_background=true`): Returns immediately with a `task_id`. System notifies via `<system-reminder>` when complete. Fire-and-forget with result collection via `background_output(task_id)`.

2. **Sync Task** (`run_in_background=false`): Blocks until task completes. Returns full result text. Has timeout handling via `syncPollTimeoutMs`.

#### BackgroundManager Architecture

The `BackgroundManager` class (`src/features/background-agent/manager.ts`) is the core orchestrator.

**Task Lifecycle:**
```
pending → running → completed/error/cancelled/interrupt
```

**Key Data Structures:**
- `tasks: Map<string, BackgroundTask>` -- All tasks
- `queuesByKey: Map<string, QueueItem[]>` -- Pending tasks by concurrency key
- `notifications: Map<string, BackgroundTask[]>` -- Pending notifications per parent
- `rootDescendantCounts: Map<string, number>` -- Tracks subagent tree depth

**Concurrency Control:** Limits concurrent tasks per model/agent via `ConcurrencyManager` to prevent resource exhaustion.

#### Task Delegation (`src/tools/delegate-task/`)

The `DelegateTask` tool requires EITHER `category` OR `subagent_type`:

```typescript
interface DelegateTaskArgs {
  load_skills: string[];        // REQUIRED
  description: string;          // Short description
  prompt: string;               // Full prompt
  run_in_background: boolean;   // REQUIRED
  category?: string;            // Category-based OR
  subagent_type?: string;       // Direct agent
  session_id?: string;          // Continue existing session
  command?: string;             // Slash command tracking
}
```

**Session Continuation**: If `session_id` is provided, the agent continues an existing session, preserving full context and saving tokens.

**Category Resolution** (`category-resolver.ts`):
- Model resolution precedence: explicit category model > sisyphus-junior override > category default > system default
- Detects unstable agents (Gemini, MiniMax, Kimi) and auto-forces them to background execution
- Supports fallback chains for retry on model errors

#### Subagent Resolution (`subagent-resolver.ts`)

For direct agent invocation:
- Validates agent exists and is callable (`mode !== "primary"`)
- Cannot call primary agents (sisyphus, atlas) via task
- Plan family agents cannot delegate to other plan agents (deadlock protection)
- Resolves model with fallback chain support

#### Notification System

When a task completes, `notifyParentSession()`:
1. Shows toast notification
2. Injects `<system-reminder>` into parent's next turn
3. Batches notifications for multiple tasks

Individual completion:
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

#### Circuit Breaker / Loop Detection

Three protection mechanisms:
1. **Tool call counting**: `task.progress.toolCalls++`, cancels if threshold exceeded
2. **Repetitive tool detection**: Tracks consecutive tool calls, cancels if threshold exceeded
3. **Stale task detection**: Tasks with no progress for extended time are interrupted

#### Spawn Limits

- **Depth limits**: `getMaxSubagentDepth(config)` prevents runaway nesting
- **Root session budget**: `getMaxRootSessionSpawnBudget(config)` limits total descendants

#### Clever Solutions

1. **Queue-based concurrency**: Tasks queued per model/agent key, processed based on concurrency limits
2. **Pre-start reservation**: `reserveSubagentSpawn()` checks limits BEFORE creating task, prevents zombie tasks
3. **Atomic task state transitions**: `tryCompleteTask()` uses atomic status check to prevent double-completion
4. **Notification batching**: Queues notifications per parent, sends single combined notification when all complete
5. **Idle deferral**: Tasks briefly idle are given grace period before marking complete, prevents premature completion
6. **Fallback chain persistence**: `setSessionFallbackChain()` persists retry chain for runtime errors

#### Technical Debt

1. Complex polling logic with extensive conditional handling for session status
2. Multiple timer maps with cleanup logic scattered across methods
3. Race condition protection via `pollingInFlight` flag -- necessary but adds complexity
4. Two-phase commit pattern for spawn limits -- `commit()` and `rollback()` must be carefully matched
5. Circular dependency in cleanup: task completion leads to notification leads to cleanup leads to session removal

---

### Feature 4: LSP + AST-Grep Tools

**Location:** `src/tools/lsp/` (38 files), `src/tools/ast-grep/` (15 files)

IDE-precision tools giving every agent professional-grade code navigation and refactoring.

#### LSP Tools

| Tool | Purpose |
|------|---------|
| `lsp_goto_definition` | Jump to symbol definition |
| `lsp_find_references` | Find all usages across workspace |
| `lsp_symbols` | Document outline or workspace symbol search |
| `lsp_diagnostics` | Errors/warnings from language server |
| `lsp_prepare_rename` / `lsp_rename` | Safe rename with preparation check |

#### LSP Architecture

```
LSP Manager
  ├── LSPClient (TypeScript)
  ├── LSPClient (Python)
  └── LSPClient (Rust)
       └── Unified LSP Client Wrapper
           - Document sync (didOpen/didChange/didSave)
           - Request/notification dispatch
           - Response parsing
```

#### Server Resolution

30+ builtin language servers: typescript, deno, vue, eslint, biome, gopls, ruby-lsp, basedpyright, pyright, ruff, rust-analyzer, clangd, svelte, astro, bash, jdtls, yaml-ls, lua-ls, php, dart, terraform, dockerfile, gleam, clojure-lsp, nixd, tinymist, haskell-language-server, kotlin-ls, ocaml-lsp, prisma, texlab, fsharp, csharp, elixir-ls, zls, sourcekit-lsp.

**Priority**: project config > user config > opencode config > builtin

**Critical Windows Fix**: Uses Node.js `child_process` spawn instead of Bun spawn on Windows due to a segfault bug (oven-sh/bun#25798). A unified process abstraction bridges Bun Subprocess and Node.js ChildProcess with an identical API.

#### AST-Grep Tools

- `ast_grep_search` -- Pattern-aware code search
- `ast_grep_replace` -- AST-aware code rewriting

**Language Support**: 25 languages via CLI (bash, c, cpp, csharp, css, elixir, go, haskell, html, java, javascript, json, kotlin, lua, nix, php, python, ruby, rust, scala, solidity, swift, typescript, tsx, yaml).

#### AST-Grep Binary Management

Auto-downloads from GitHub releases. Platform-specific caching in:
- Linux/macOS: `~/.cache/oh-my-openagent/bin/`
- Windows: `%LOCALAPPDATA%/oh-my-openagent/bin/`

**Two-pass rewrite pattern**: The ast-grep CLI silently ignores `--update-all` when `--json` is present. The code works around this by running two separate invocations: first with `--json=compact` to get match results, then with `--update-all` to actually write files.

#### Clever Solutions

- **Dual-result handling**: LSP can return `Location | Location[] | LocationLink[]`; code handles all three
- **Graceful degradation**: Diagnostics endpoint falls back to cached diagnostics if `textDocument/diagnostic` fails
- **Automatic binary fallback chain**: PATH > @ast-grep/cli > download > error with install options
- **Recursive retry on ENOENT**: Auto-downloads if binary missing

#### Technical Debt

- 1-second hardcoded sleep after file open (arbitrary but functional)
- No streaming/chunking for large symbol results
- Directory diagnostics runs sequentially per file
- Default timeout of 300 seconds (5 minutes) is quite long for ast-grep searches

---

### Feature 5: Built-in MCPs

**Location:** `src/mcp/` (10 files)

Three built-in MCP servers always available without manual configuration.

#### Three Built-in MCPs

| MCP | Provider | Purpose | Auth |
|-----|----------|---------|------|
| **websearch** | Exa (default) or Tavily | Real-time web search | `EXA_API_KEY` or `TAVILY_API_KEY` |
| **context7** | context7.com | Official docs lookup for any library/framework | `CONTEXT7_API_KEY` |
| **grep_app** | grep.app | Ultra-fast code search across public GitHub repos | None (public) |

#### Websearch Detail

Default provider is Exa, with Tavily as an alternative:

```
Exa: https://mcp.exa.ai/mcp?tools=web_search_exa
     + ?exaApiKey=...&x-api-key: ... (if EXA_API_KEY set)

Tavily: https://mcp.tavily.com/mcp/
     + Authorization: Bearer {TAVILY_API_KEY}
```

**Clever detail**: API keys are URL-encoded in the Exa URL to handle special characters like `+`, `&`, `=`.

#### Context7 Detail

```typescript
const context7 = {
  type: "remote" as const,
  url: "https://mcp.context7.com/mcp",
  enabled: true,
  headers: process.env.CONTEXT7_API_KEY
    ? { Authorization: `Bearer ${process.env.CONTEXT7_API_KEY}` }
    : undefined,
  oauth: false as const,  // Disable OAuth auto-detection
}
```

`oauth: false` is critical -- it explicitly disables OAuth auto-detection since Context7 uses API key auth, not OAuth.

#### Skill-Embedded MCPs

Skills can carry their own on-demand MCP servers via `src/tools/skill-mcp/` and `src/hooks/skill-mcp-manager/`:
- Spin up scoped to current task
- Disappear when task completes
- Keep context window clean

#### MCP Creation Factory

Each MCP can be disabled individually via `disabledMcps` array in `createBuiltinMcps()`.

#### Technical Debt / Issues

1. Context7's `CONTEXT7_API_KEY` vs `Bearer` mismatch in code comment (comment says Bearer but key name implies API key style)
2. No retry logic for MCP connection failures
3. No connection pooling for multiple MCP requests

---

### Feature 6: ultrawork / Ralph Loop

**Location:** `src/hooks/ralph-loop/` (27 files)

A self-referential development loop where the agent works continuously toward a goal, detecting `<promise>DONE</promise>` for completion. If the agent stops without completing, the system yanks it back. Ultrawork is an enhanced version with Oracle verification.

#### Activation

The `keyword-detector` hook monitors for `ultrawork` or `ulw` (case insensitive, with word boundary detection). Patterns in code blocks are ignored to prevent false triggers.

#### State Management

State persists in `.sisyphus/ralph-loop.local.md` as YAML frontmatter:

```yaml
---
active: true
iteration: 1
max_iterations: 100
completion_promise: "DONE"
started_at: "2026-03-26T..."
session_id: "session-uuid"
ultrawork: false
strategy: "continue"
---
Original task prompt goes here
```

#### Completion Detection

Two mechanisms:

1. **Transcript Scan (Primary)**:
   - Reads `.opencode/transcripts/{session_id}.jsonl`
   - Filters for user messages with timestamp filtering (only messages after `started_at`)
   - Regex: `<promise>{PROMISE}</promise>` with `s` flag (dotall)

2. **API Fallback**:
   - Calls `ctx.client.session.messages()` API
   - Extracts text from assistant messages only
   - Scoped to messages since `message_count_at_start`

#### Loop Flow (on `session.idle`)

```
session.idle event
    ├── Completion detected? ──→ Handle complete
    ├── Verification pending? ──→ Handle pending verification ──→ Increment iteration ──→ Continue loop
    └── Max iterations reached? ──→ Clear & stop
```

#### Iteration Continuation

Two strategies:
- **`continue` (default)**: Injects continuation prompt into same session
- **`reset`**: Creates new child session, injects there, switches TUI to new session

**Continuation prompt**:
```
[SYSTEM DIRECTIVE] - RALPH LOOP {ITERATION}/{MAX}]

Your previous attempt did not output the completion promise. Continue working on the task.

IMPORTANT:
- Review your progress so far
- Continue from where you left off
- When FULLY complete, output: <promise>{PROMISE}</promise>
- Do not stop until the task is truly done
```

#### Ultrawork Verification (Oracle)

When `<promise>DONE</promise>` is detected in ultrawork mode:

1. Phase 1: Changes `completion_promise` to "VERIFIED", injects Oracle verification prompt
2. Phase 2: System monitors for Oracle's `<promise>VERIFIED</promise>`
3. Phase 3: If verified -- loop ends. If not verified -- restart with `VERIFICATION FAILED` prompt including Oracle's concerns

#### Clever Solutions

1. **Completion tag regex with dotall**: `<promise>.*?</promise>` with `s` flag matches across lines
2. **Timestamp filtering**: Only considers messages after loop started, ignoring historical context
3. **Message scoping**: Uses `message_count_at_start` to scope search to loop-relevant messages
4. **Session hierarchy**: Child sessions inherit context but track separately
5. **Two-phase ultrawork**: Separates "done" from "verified" with Oracle as final arbiter
6. **Tool inheritance**: Continuation preserves tool permissions from parent session
7. **Transcript fallback**: If API fails, falls back to direct transcript file reading

#### Technical Debt

- Hardcoded 5-second API timeout may be too short for large contexts
- State file polling (no event-driven updates)
- No graceful degradation if Oracle agent unavailable
- File-based state not atomic (potential race conditions in concurrent access)

---

### Feature 7: Todo Enforcer & Task System

**Location:** `src/tools/task/` (15 files), `src/hooks/todo-continuation-enforcer/` (30+ files)

A file-based task system with full dependency management. Tasks persist in `.opencode/tasks/{listId}/` with atomic JSON writes.

#### Task Schema

```typescript
TaskObject {
  id: string,           // "T-{uuid}"
  subject: string,
  description: string,
  status: "pending" | "in_progress" | "completed" | "deleted",
  activeForm?: string,
  blocks: string[],      // Task IDs this blocks
  blockedBy: string[],   // Task IDs blocking this
  owner?: string,
  metadata?: Record<string, unknown>,
  repoURL?: string,
  parentID?: string,
  threadID: string,       // OpenCode session ID
}
```

#### Dependency Management

- When creating: specify `blockedBy: ["T-xxx", "T-yyy"]` to declare dependencies
- When updating: use `addBlockedBy: [...]` to add without replacing
- TaskList filters `blockedBy` at query time (not storage) to always show current state
- Tasks with empty `blockedBy` can run concurrently

#### Storage Layer

Atomic writes via temp file rename:
```typescript
writeJsonAtomic(filePath, data):
  1. Write to ${filePath}.tmp.{timestamp}
  2. renameSync(tempPath, filePath)  // Atomic on POSIX
```

Lock acquisition with stale detection: Creates `.lock` file with UUID + timestamp, detects stale locks (30s threshold), only one process can hold lock at a time.

#### Todo Continuation Enforcer

Yanks idle agents back to work if they stop with incomplete tasks. The idle handler checks 14 conditions before injecting a continuation:

1. Skip if `isRecovering` flag set
2. Skip if abort detected within 3s window
3. Skip if background tasks running
4. Skip if last assistant message was aborted (API fallback)
5. Skip if pending question awaiting user response
6. Skip if no todos or all todos complete
7. Skip if injection already in flight
8. Skip if consecutive failures >= 5 within 5min window
9. Skip if cooldown period not elapsed (exponential backoff: 5s * 2^failures)
10. Skip if compaction guard still armed (60s after session.compacted)
11. Skip if agent in skipAgents list (prometheus, compaction, plan)
12. Track progress: if incompleteCount decreased or todo snapshot changed -> reset stagnation
13. If stagnationCount >= 3 -> stop continuation
14. Otherwise start countdown (2s) then inject continuation prompt

**Stagnation detection**: Tracks `lastIncompleteCount` and `lastTodoSnapshot`. Progress = incompleteCount decreased OR completed count increased OR todo snapshot changed. After successful injection, waits for next idle to check if progress made.

**Exponential backoff**: `cooldown = 5s * 2^failures` prevents spam while allowing recovery.

#### Clever Solutions

1. **Query-time blockedBy filtering**: Always shows current state
2. **Exponential backoff**: Prevents spam while allowing recovery
3. **Compaction guard**: 60s window after session.compacted prevents immediate re-intervention
4. **Abort window**: 3s handles API retry scenarios gracefully
5. **Progress reset**: Stagnation count resets when real progress is made

#### Technical Debt

- Session state store has TTL-based pruning (10min entries, 2min prune interval) but no max size limit
- Lock file cleanup relies on `isStale()` check running on every acquire attempt
- `syncAllTasksToTodos` has complex filtering logic that could miss edge cases with id-less todos

---

## Tier 2: Secondary Features

---

### Feature 8: Category-Based Task Delegation

**Location:** `src/tools/delegate-task/category-resolver.ts`, `src/shared/session-category-registry.ts`, `src/agents/sisyphus-junior/`

Categories are agent configuration presets that optimize model selection for specific task domains. Instead of specifying a model directly, delegations use a category name.

#### Default Categories

```typescript
DEFAULT_CATEGORIES = {
  "visual-engineering": { model: "google/gemini-3.1-pro", variant: "high" },
  ultrabrain: { model: "openai/gpt-5.4", variant: "xhigh" },
  deep: { model: "openai/gpt-5.3-codex", variant: "medium" },
  artistry: { model: "google/gemini-3.1-pro", variant: "high" },
  quick: { model: "openai/gpt-5.4-mini" },
  "unspecified-low": { model: "anthropic/claude-sonnet-4-6" },
  "unspecified-high": { model: "anthropic/claude-opus-4-6", variant: "max" },
  writing: { model: "kimi-for-coding/k2p5" },
}
```

#### Category Prompt Appends

Each category adds domain-specific instructions:
- `visual-engineering`: Mandates design system analysis before coding
- `ultrabrain`: Deep reasoning, architectural decisions, strategic advisor mindset
- `deep`: Autonomous problem-solving, goal-oriented, minimal check-ins
- `quick`: Efficient execution, smaller model, requires exhaustive explicitness
- `writing`: Anti-AI-slop rules (no em-dashes, no filler phrases, plain words)

#### Sisyphus-Junior Agent

Route-based prompt builder for different model families:
- GPT models -> `gpt.ts` / `gpt-5-4.ts` / `gpt-5-3-codex.ts`
- Gemini models -> `gemini.ts`
- Default (Claude) -> `default.ts`

**Blocked tools**: `["task"]` -- Sisyphus-Junior cannot spawn other agents (prevents infinite recursion).

#### Category-Skill Reminder Hook

Reminds orchestrator agents about the category+skill system after 3+ tool calls without using delegation tools. Tracks per-session state and clears on session deletion or compaction.

#### Technical Debt

- Custom category override gap: If user overrides model, static prompt appends become misleading
- Unknown categories produce error but disabled categories silently return null

---

### Feature 9: Built-in Skills

**Location:** `.opencode/skills/`, `src/tools/skill/`, `src/features/opencode-skill-loader/` (35+ files)

Pre-packaged skill workflows with optional embedded MCP servers. Skills provide specialized knowledge and step-by-step guidance for specific domains.

#### Skill Structure

```markdown
---
name: github-triage
description: "Read-only GitHub triage for issues AND PRs..."
model: ...  # optional override
agent: ...  # optional restriction
mcp: ...    # optional embedded MCP config
---

# GitHub Triage - Read-Only Analyzer

<role>
Read-only GitHub triage orchestrator...
</role>

<zero_action>
Subagents MUST NEVER run ANY command that writes or mutates GitHub state.
</zero_action>
```

#### Skill Loading

**Discovery paths** (priority high to low):
1. Project: `{cwd}/.opencode/skills/`
2. User: `~/.config/opencode/skills/`
3. OpenCode builtin: Built-in plugin skills
4. Plugin: From plugins

**Skill scopes** (priority high to low):
```typescript
scopePriority = {
  project: 4,
  user: 3,
  opencode: 2,
  "opencode-project": 2,
  plugin: 1,
  config: 1,
  builtin: 1,
}
```

#### Available Skills

- **github-triage**: Read-only GitHub triage, one issue/PR per background quick subagent, zero-action policy
- **work-with-pr**: Working with pull requests
- **pre-publish-review**: Pre-publish review
- **work-with-pr-workspace**: PR workspace support

#### Skill-Embedded MCPs

Skills can specify `mcp` config for on-demand MCP servers:
- Spin up scoped to task
- Disappear when done
- Keep context window clean

#### Clever Solutions

1. **Lazy content loading**: Uses `lazyContent.load()` to defer loading until needed
2. **Scope priority system**: Higher priority scopes override lower ones
3. **Skill-embedded MCP**: MCP servers only spin up when skill is invoked

#### Technical Debt

- Cache invalidation: `clearSkillCache()` called before `getAllSkills()` but concurrent calls could get stale cache
- Agent restrictions: Hard failure if agent mismatch, no fallback or warning
- Circular skill dependencies: No detection possible, could cause infinite loops

---

### Feature 10: Deep Initialization (/init-deep)

**Location:** `.opencode/command/`, `src/hooks/directory-agents-injector/` (9 files)

Generates hierarchical `AGENTS.md` files throughout the project. Agents auto-read relevant context automatically.

#### init-deep Template

Sophisticated template-driven generator:
1. Discovery + Analysis (concurrent): Fire background explore agents immediately
2. Score & Decide: Determine AGENTS.md locations from merged findings
3. Generate: Root first, then subdirs in parallel
4. Review: Deduplicate, trim, validate

**Scoring matrix:**

| Factor | Weight | High Threshold | Source |
|--------|--------|----------------|--------|
| File count | 3x | >20 | bash |
| Subdir count | 2x | >5 | bash |
| Code ratio | 2x | >70% | bash |
| Module boundary | 2x | Has index.ts/__init__.py | bash |
| Symbol density | 2x | >30 symbols | LSP |
| Reference centrality | 3x | >20 refs | LSP |

**Decision rules:**
- Root (.) -- ALWAYS create
- Score >15 -- Create AGENTS.md
- Score 8-15 -- Create if distinct domain
- Score <8 -- Skip (parent covers)

#### directory-agents-injector Hook

Automatically injects AGENTS.md content when files are read:
- `findAgentsMdUp()` walks up from the read file's directory toward root
- Root AGENTS.md is intentionally skipped because OpenCode's system.ts already loads it
- Content is truncated via `dynamicTruncator` to save context
- Caches injected paths per session to avoid duplicate injection

**Issue #379**: Root AGENTS.md was being injected twice before the fix -- now explicitly skipped in the hook.

#### Technical Debt

- Silent failures: `processFilePathForAgentsInjection` catches errors with empty catch block
- Session cache persistence: Uses file-based storage which could fail mid-session

---

### Feature 11: Prometheus Strategic Planner

**Location:** `src/agents/prometheus/` (12 files), `src/agents/metis.ts`

An interview-mode strategic planner. Before touching code, Prometheus questions the user, identifies scope and ambiguities, and builds a verified plan.

#### Two-Phase Workflow

**Phase 1: Interview Mode (DEFAULT)**
- Intent classification first (Trivial, Refactoring, Build from Scratch, Mid-sized, Collaborative, Architecture, Research)
- Different interview strategies per intent type
- Research agents fire in parallel (explore, librarian)
- Draft file continuously updated at `.sisyphus/drafts/{name}.md`
- Clearance check after every turn to auto-transition to Phase 2

**Phase 2: Plan Generation (AUTO-TRANSITION)**
Triggered when ALL clearance checklist items pass:
- Core objective clearly defined
- Scope boundaries established (IN/OUT)
- No critical ambiguities remaining
- Technical approach decided
- Test strategy confirmed
- No blocking questions outstanding

#### Critical Constraints

1. **INTERVIEW MODE BY DEFAULT**: Consultant first, planner second
2. **AUTOMATIC PLAN GENERATION**: Self-clearance check after every turn
3. **MARKDOWN-ONLY FILE ACCESS**: Enforced by `prometheus-md-only` hook
4. **STRICT PATH ENFORCEMENT**: Only `.sisyphus/plans/*.md` and `.sisyphus/drafts/*.md`
5. **MAXIMUM PARALLELISM PRINCIPLE**: 5-8 tasks per wave, one task = one module
6. **SINGLE PLAN MANDATE**: Everything in ONE plan, never split
7. **INCREMENTAL WRITE PROTOCOL**: Write skeleton + Edit batches to avoid output limits
8. **DRAFT AS WORKING MEMORY**: Update after every meaningful exchange

#### prometheus-md-only Hook

**Blocked tools**: `edit`, `write`, `apply_patch` for non-.md files

**Allowed paths ONLY**:
- `.sisyphus/plans/{name}.md`
- `.sisyphus/drafts/{name}.md`

**Blocked paths**: `docs/`, `plan/`, `plans/`, any path outside `.sisyphus/`

#### Metis Pre-Planning Consultant

Named after the Greek goddess of wisdom. Called by Prometheus BEFORE generating plans.

**Key responsibilities:**
- Identify hidden intentions and unstated requirements
- Detect ambiguities that could derail implementation
- Flag AI-slop patterns (over-engineering, scope creep)
- Generate clarifying questions for the user
- Prepare directives for the planner agent

#### Model-Specific Prompts

- **Claude (default)**: Modular sections (identity + interview + plan-gen + high-accuracy + template + behavioral)
- **GPT**: XML-tagged, principle-driven format
- **Gemini**: Aggressive tool-call enforcement, thinking checkpoints

#### Technical Debt

- Complex prompt composition: 6 different files combined into one system prompt
- Silent failure in Metis consultation: If Metis task fails, plan generation continues anyway
- Output limit vulnerability: Despite incremental write protocol, very large plans could still stall

---

### Feature 12: Context Window Management & Session Compaction

**Location:** `src/hooks/preemptive-compaction.ts`, `src/hooks/context-window-monitor.ts`, `src/hooks/compaction-context-injector/` (16 files), `src/shared/dynamic-truncator.ts`

Proactive session compaction that preserves critical context before hitting token limits.

#### Context Window Monitor

**Key threshold**: `CONTEXT_WARNING_THRESHOLD = 0.70` (70%)

Tracks token usage via `message.updated` events, displays usage percentage in tool output when >70%.

#### Preemptive Compaction

**Key threshold**: `PREEMPTIVE_COMPACTION_THRESHOLD = 0.78` (78%)

Triggers session summarization BEFORE hitting limits, not when already at limit.

Flow:
1. Track token usage via `message.updated` events (cached in `tokenCache`)
2. On `tool.execute.after`: check usage ratio
3. If >78%, trigger `summarize()` via session API
4. 120-second timeout on compaction operation

#### Dynamic Truncator

Two key functions:

**`truncateToTokenLimit()`**:
- Estimates tokens via `chars / 4` (rough approximation)
- Preserves header lines (default 3)
- Adds truncation notice: `[X more lines truncated due to context window limit]`

**`dynamicTruncate()`**:
- Gets actual context window usage via `getContextWindowUsage()`
- Calculates max output tokens as `min(remainingTokens * 0.5, targetMaxTokens)`
- If remaining < 0: returns `[Output suppressed - context window exhausted]`

#### Compaction Context Injector

Preserves critical context during compaction:
- `capture()`: Before compaction, saves agent config checkpoint
- `inject()`: After compaction, injects recovery context

Handles `session.compacted`, `session.deleted`, `session.idle`, `message.updated`, `message.part.delta`, and `message.part.updated` events.

#### Clever Solutions

1. **Preemptive not reactive**: Compacts before hitting limit, not when already at limit
2. **Degradation monitoring**: Tracks if compaction causes quality degradation
3. **No-text-tail detection**: Catches sessions where assistant produces empty responses
4. **Context preservation**: Agent config checkpoint ensures compaction doesn't lose agent identity
5. **Streaming-aware**: Tracks `message.part.delta` events for real-time output monitoring

#### Technical Debt

- Token estimation is rough: `chars / 4` simplification -- actual tokenizers vary
- Silent failures: Many operations catch and log errors but continue silently
- Cache persistence: Session caches stored in memory -- lost on crash
- 120-second timeout on compaction could leave session in inconsistent state if summarize hangs

---

### Feature 13: Session Recovery & Model Fallback

**Location:** `src/hooks/session-recovery/` (20 files), `src/hooks/runtime-fallback/` (30 files), `src/hooks/model-fallback/`, `src/shared/model-error-classifier.ts`

Three distinct recovery mechanisms.

#### Session Recovery

Handles specific error patterns with surgical fixes:
- `tool_result_missing`: Tool use without corresponding result -- injects "Operation cancelled" tool results
- `unavailable_tool`: Model called tool that doesn't exist
- `thinking_block_order`: Thinking blocks in wrong order
- `thinking_disabled_violation`: Thinking block used when disabled
- `assistant_prefill_unsupported`: Prefill not supported (non-recoverable)

If `experimental.auto_resume` is enabled, auto-resumes the session after recovery.

#### Runtime Fallback

Automatic model switching on retryable API errors:

**Retryable errors**: Status codes 429, 503, 529; error names like `ratelimiterror`, `quotaexceedederror`, `modelunavailableerror`, `providerconnectionerror`, `authenticationerror`, etc.

**Non-retryable errors**: `messageabortederror`, `permissiondeniederror`, `contextlengtherror`, `timeouterror`, `validationerror`, `syntaxerror`, `usererror`

Flow:
1. `session.error` event fires
2. `isRetryableError()` check
3. If yes, extract status code, get fallback models
4. `prepareFallback()` selects next model in chain
5. `promptAsync()` with new model + retry parts

#### Model Fallback

Fallback chain management when primary models are unavailable:

```typescript
type FallbackEntry = {
  providers: string[];
  model: string;
  variant?: string;
  reasoningEffort?: string;
  temperature?: number;
  top_p?: number;
  maxTokens?: number;
  thinking?: { type: "enabled" | "disabled"; budgetTokens?: number };
};
```

**Sisyphus fallback chain** (7 entries):
1. `anthropic/github-copilot/opencode` + `claude-opus-4-6` (variant: max)
2. `opencode-go` + `kimi-k2.5`
3. `kimi-for-coding` + `k2p5`
4. Multiple providers + `kimi-k2.5`
5. `openai/github-copilot/opencode` + `gpt-5.4` (variant: medium)
6. `zai-coding-plan/opencode` + `glm-5`
7. `opencode` + `big-pickle`

**Key insight**: Provider connectivity is checked (not model list) because model lists can be stale (users manually add models).

#### Technical Decisions

1. No-op fallback skip: Skips where provider/model identical to current
2. Provider-first model lists: Only provider connectivity checked for reachability
3. Attempt count preservation: `attemptCount` preserved across `session.error` retries
4. Module-level state: `pendingModelFallbacks`, `lastToastKey`, `sessionFallbackChains` are module-level Maps (singleton)

---

### Feature 14: Claude Code Compatibility Layer

**Location:** `src/plugin-handlers/` (27 files), `src/hooks/claude-code-hooks/`

Full compatibility for Claude Code hooks, commands, skills, agents, MCPs, and plugins. Supports loading from `~/.claude/`, `.claude/`, and Claude Code's `settings.json` hooks system.

#### Configuration Flow

```
createConfigHandler()
  ├── applyProviderConfig() --> Configure providers
  ├── loadPluginComponents() --> Load builtin plugin components
  ├── applyAgentConfig() --> Merge agents from all sources
  ├── applyToolConfig() --> Configure tools
  ├── applyMcpConfig() --> Merge MCP servers
  └── applyCommandConfig() --> Merge commands and skills
```

#### Agent Loading Priority

1. Builtin agents
2. Sisyphus configuration (if enabled): Sisyphus + Sisyphus-Junior + Prometheus + Atlas
3. Filtered user agents (from `~/.claude/`)
4. Filtered project agents (from `.claude/`)
5. Filtered plugin agents (from plugin components)
6. Custom config agents (from opencode.json)

**Protected agents**: sisyphus, sisyphus-junior, prometheus, OpenCode-Builder, atlas, and all builtin agents cannot be overridden by user/project/agent configs.

#### Skill Discovery Order

1. Config source skills
2. OpenCode project skills
3. Project Claude skills (`.claude/skills`)
4. Project agents skills (`.claude/agents`)
5. OpenCode global skills
6. User Claude skills (`~/.claude/skills`)
7. Global agents skills (`~/.claude/agents`)

#### Claude Code Hooks Bridge

```typescript
createClaudeCodeHooksHook(ctx, config, contextCollector)
  ├── pre-compact
  ├── chat.message
  ├── tool.execute.before
  ├── tool.execute.after
  └── session.event
```

#### Technical Decisions

1. **Parallel discovery**: All skill/agent/command discovery runs in `Promise.all()` for speed
2. **Agent key remapping**: Maps Claude Code agent names to Oh-My-OpenCode display names via `AGENT_NAME_MAP`
3. **Priority ordering**: `reorderAgentsByPriority()` ensures proper agent ordering after merge
4. **Prometheus config builder**: Special handling to build Prometheus agent with proper overrides

---

### Feature 15: Session Notifications & Auto-Update Checking

**Location:** `src/hooks/session-notification.ts` (27 files), `src/hooks/background-notification/`, `src/hooks/auto-update-checker/`

OS-native notifications when background agents complete or agents go idle. Works on macOS, Linux, and Windows.

#### Session Notification Architecture

**Platform detection**:
- macOS: `terminal-notifier` first (deterministic click-to-focus), fallback to `osascript`
- Linux: `notify-send`
- Windows: PowerShell toast via `BurntToast`

#### Idle Notification Scheduler

```typescript
{
  idleConfirmationDelay: 1500,  // Wait before notifying
  skipIfIncompleteTodos: true,
  maxTrackedSessions: 100,
  enforceMainSessionFilter: true,
  activityGracePeriodMs: 100,
}
```

On `session.idle`: Schedule notification after delay. If activity occurs before delay, cancel. `activityGracePeriodMs` ignores late-arriving activity events.

#### Notification Types

1. **Idle**: "Agent is ready for input"
2. **Question**: "Agent is asking a question" (when `question` tool called)
3. **Permission**: "Agent needs permission to continue"

#### Background Notification Hook

Notifications delivered directly via `session.prompt({ noReply })` from `BackgroundManager`, so the hook only handles event routing.

#### Auto-Update Checker

On first session.created (non-child session):
1. Get cached version from npm
2. Get local dev version if running from source
3. Show version toast: "OpenCode is now on Steroids..."

Child sessions (background/child) are skipped.

#### Clever Solutions

1. **Grace period for activity**: Prevents notification cancellation from race conditions with late-arriving events
2. **Platform detection caching**: Cached at creation time, not per-event
3. **Subagent filtering**: Background agent sessions filtered out
4. **Main session enforcement**: Optional filter ensures only main session generates notifications

---

### Feature 16: Comment Checker & Quality Hooks

**Location:** `src/hooks/comment-checker/` (12 files), `src/hooks/thinking-block-validator/`, `src/hooks/edit-error-recovery/`, `src/hooks/write-existing-file-guard/`

Four distinct hooks maintaining code quality and preventing common AI agent mistakes.

#### 1. Comment Checker

Uses `@code-yeongyu/go-claude-code-comment-checker` (Go binary) for actual comment analysis. Lazy-downloads binary from GitHub Releases on first use.

**Hook points**: `tool.execute.before` (registers pending calls) and `tool.execute.after` (runs comment checker CLI on success).

**Concurrency control**: Semaphore pattern via `withCommentCheckerLock()` -- only one check runs at a time.

**Smart filtering**: Intelligently ignores BDD tests, directives, and docstrings.

#### 2. Thinking Block Validator

Prevents "Expected thinking/redacted_thinking but found tool_use" API errors by validating message structure **before** sending to Anthropic API.

**Hook point**: `experimental.chat.messages.transform` -- Called before messages are converted to ModelMessage format.

**Key logic**: If an assistant message has `tool_use` but doesn't start with a thinking block, prepend the most recent valid signed thinking block from message history.

**Signature validation**: Only real `type: "thinking"` blocks with valid `signature` field trigger the hook. GPT `type: "reasoning"` blocks are excluded (no Anthropic signature).

**Critical constraint**: Cannot inject synthetic thinking blocks -- only reuse existing signed blocks verbatim to avoid signature validation failures.

#### 3. Edit Error Recovery

Detects common Edit tool errors and injects a recovery reminder:

**Error patterns detected**:
- "oldString and newString must be different"
- "oldString not found"
- "oldString found multiple times"

**Recovery reminder**: Forces agent to READ the file immediately, verify content, apologize, then continue with corrected action.

#### 4. Write Existing File Guard

Prevents agents from overwriting existing files without first reading them.

**Permission model**:
- `tool.execute.before` on `read` grants write permission
- `tool.execute.before` on `write` consumes write permission
- Permission is session-scoped and path-canonicalized

**Blocked**: Write to existing file without `overwrite: true` flag and without prior `read` in same session.

**Allowed**: Non-existing files, `overwrite: true`, `.sisyphus/**` files, files outside session directory.

**LRU eviction**: When sessions exceed 256, least recently used session is evicted.

**Path normalization**: Uses `realpathSync.native` to resolve symlinks and normalize casing.

#### Test Coverage

The write-existing-file-guard has 18 test scenarios covering non-existing files, existing without read, same-session read-then-write, concurrent writes, cross-session invalidation, overwrite flag variants, path normalization, symlinks, and LRU eviction.

---

## Cross-Feature Interactions

### Hashline + Background Agents
Edit operations run in main session; background agents may read files with hashline markers. No direct coupling -- hashline is transparent to callers.

### Multi-Agent Orchestration + Background Agents
Sisyphus delegates to subagents via `task()` tool. `task(run_in_background=true)` spawns background agents. Session hierarchy: parent creates child sessions. Sisyphus prompt explicitly instructs parallel delegation.

### Multi-Agent Orchestration + Hashline
Sisyphus uses `edit` tool extensively. Hashline validation prevents stale edits. Error recovery includes re-reading and re-editing.

### Session Recovery + Model Fallback
If session recovery fails and `experimental.auto_resume` is enabled, it triggers a new session prompt which could invoke model fallback if configured.

### Claude Code Compatibility + Session Notifications
The Claude Code hooks system provides the bridge that allows session events (idle, error, etc.) to flow to the notification system.

### Background Notifications + Session Notifications
Background agent completion notifications flow through `BackgroundManager` which integrates with the session notification scheduler.

### Runtime Fallback + Model Fallback
Runtime fallback handles transient API errors with retry; model fallback handles persistent unavailability through fallback chain progression.

---

## Notable Technical Patterns

### Concurrency Management
Multiple systems use queue-based concurrency with semaphore patterns. Background tasks, comment checking, and session notifications all implement their own concurrency control rather than sharing a common abstraction.

### File-Based State Persistence
Ralph Loop, init-deep, and the task system all use file-based state that persists beyond session boundaries. This enables cross-session continuity but introduces potential race conditions and consistency challenges.

### Model-Specific Prompt Variants
Throughout the codebase, model-specific handling appears via string replacement, conditional sections, and route-based prompt builders. GPT, Gemini, and Claude each receive tailored prompts. This is a recurring architectural pattern that appears in Sisyphus, Prometheus, Sisyphus-Junior, and the dynamic agent prompt builder.

### Hook-Driven Architecture
The entire plugin is structured around hooks -- event handlers at key lifecycle points. Tools, agents, MCPs, and features all communicate through this hook system rather than direct coupling.

### Lazy Binary Downloads
Comment checking and ast-grep both lazy-download Go/binaries from GitHub Releases on first use. This avoids bundling large binaries but requires network access and version management.

---

## Summary Table

| # | Feature | Priority | LOC Estimate | Key Pattern |
|---|---------|----------|--------------|-------------|
| 1 | Multi-Agent Orchestration | CORE | ~2000 | Factory + Strategy |
| 2 | Hash-Anchored Edit Tool | CORE | ~3000 | Hash table + validation |
| 3 | Background Agents | CORE | ~4000 | Queue + event + polling |
| 4 | LSP + AST-Grep | CORE | ~2000 | LSP client wrapper |
| 5 | Built-in MCPs | CORE | ~500 | Remote MCP factory |
| 6 | ultrawork / Ralph Loop | CORE | ~2000 | State machine + event |
| 7 | Todo Enforcer | CORE | ~2500 | File-based + atomic writes |
| 8 | Category Delegation | SECONDARY | ~1500 | Config resolution |
| 9 | Built-in Skills | SECONDARY | ~2000 | Template loader |
| 10 | Deep Initialization | SECONDARY | ~1000 | Scoring matrix |
| 11 | Prometheus Planner | SECONDARY | ~2000 | Interview + plan gen |
| 12 | Context Compaction | SECONDARY | ~1500 | Token tracking + truncation |
| 13 | Session Recovery | SECONDARY | ~2500 | Error classification |
| 14 | Claude Code Compat | SECONDARY | ~2000 | Config merge |
| 15 | Notifications | SECONDARY | ~1500 | Platform abstraction |
| 16 | Comment Checker | SECONDARY | ~1000 | External CLI + hooks |
