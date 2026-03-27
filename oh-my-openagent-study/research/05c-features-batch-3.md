# Features Batch 3 Research (Features 7-9)

## Feature 7: Todo Enforcer & Task System

### Core Implementation

**File-based task storage** with atomic JSON writes. Tasks persist in `.opencode/tasks/{listId}/` directory.

**Key Files:**
- `src/tools/task/types.ts` - Zod schemas for TaskObject, TaskCreateInput, TaskUpdateInput
- `src/tools/task/task-create.ts` - Task creation with auto-generated T-{uuid} IDs
- `src/tools/task/task-list.ts` - Lists active tasks, filters blockedBy to unresolved blockers only
- `src/tools/task/task-update.ts` - Updates with addBlocks/addBlockedBy for dependency management
- `src/tools/task/todo-sync.ts` - Syncs task state to OpenCode's Todo API
- `src/features/claude-tasks/storage.ts` - File I/O with atomic writes, lock acquisition

**Task Schema:**
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

### Dependency Management

**Task linking via blockedBy/blocks:**
- When creating: specify `blockedBy: ["T-xxx", "T-yyy"]` to declare dependencies
- When updating: use `addBlockedBy: [...]` to add without replacing
- TaskList filters blockedBy to only show unresolved (non-completed) blockers

**Parallel execution:** Tasks with empty blockedBy can run concurrently. TaskList reminder:
> "1 task = 1 task. Maximize parallel execution by running independent tasks (tasks with empty blockedBy) concurrently."

### Storage Layer

**Atomic writes via temp file rename:**
```typescript
writeJsonAtomic(filePath, data):
  1. Write to ${filePath}.tmp.{timestamp}
  2. renameSync(tempPath, filePath)  // Atomic on POSIX
```

**Lock acquisition with stale detection:**
- Creates `.lock` file with UUID + timestamp
- Detects stale locks (30s threshold) and removes them
- Only one process can hold lock at a time

### Todo Continuation Enforcer Hook

**Purpose:** Yanks idle agents back to work if they stop with incomplete tasks.

**Key Files:**
- `src/hooks/todo-continuation-enforcer/` - 30+ files including tests
- `src/hooks/todo-continuation-enforcer/index.ts` - Exports createTodoContinuationEnforcer
- `src/hooks/todo-continuation-enforcer/handler.ts` - Event handler for session.idle, session.error, session.compacted
- `src/hooks/todo-continuation-enforcer/idle-event.ts` - Core logic for handling idle sessions
- `src/hooks/todo-continuation-enforcer/session-state.ts` - Tracks stagnation count, failures, countdowns per session

**Session Idle Handling Flow (idle-event.ts):**
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

**Stagnation Detection:**
- Tracks `lastIncompleteCount` and `lastTodoSnapshot`
- Progress = incompleteCount decreased OR completed count increased OR todo snapshot changed
- After successful injection, waits for next idle to check if progress made
- stagnationCount increments only after a successful injection that didn't yield progress
- MAX_STAGNATION_COUNT = 3

**Continuation Prompt (constants.ts):**
```
Incomplete tasks remain in your todo list. Continue working on the next pending task.
- Proceed without asking for permission
- Mark each task complete when finished
- Do not stop until all tasks are done
- If you believe all work is already complete, the system is questioning your completion claim.
  Critically re-examine each todo item from a skeptical perspective...
```

### Compaction Todo Preserver

**Purpose:** Preserves todo state across session compaction events.

**Flow:**
1. On `session.compacted`: restore todos from in-memory snapshot
2. On `session.deleted`: clear snapshot

**Implementation:** `src/hooks/compaction-todo-preserver/hook.ts`
- Captures todo snapshot on demand via `capture()` method
- Stores snapshots in Map<sessionID, TodoSnapshot[]>
- On compaction event, restores from snapshot if no current todos exist

### Tasks TodoWrite Disabler

**Purpose:** Disables native TodoWrite/TodoRead tools when experimental.task_system is enabled.

**Blocked tools:** `["TodoWrite", "TodoRead"]`

**Error message when blocked:**
```
TodoRead/TodoWrite are DISABLED because experimental.task_system is enabled.

**ACTION REQUIRED**: RE-REGISTER what you were about to write as Todo using Task tools NOW.

**Workflow:**
1. TaskCreate({ subject: "your task description" })
2. TaskUpdate({ id: "T-xxx", status: "in_progress" })
3. DO THE WORK
4. TaskUpdate({ id: "T-xxx", status: "completed" })

CRITICAL: 1 task = 1 task. Fire independent tasks concurrently.
```

### Clever Solutions / Technical Debt

**Clever:**
- Task list filters blockedBy at query time (not storage) to always show current state
- Exponential backoff on continuation cooldown prevents spam while allowing recovery
- Compaction guard prevents continuation immediately after compaction (60s window)
- Abort window (3s) handles API retry scenarios gracefully

**Technical Debt / Edge Cases:**
- Session state store has TTL-based pruning (10min entries, 2min prune interval) but no max size limit - could grow unbounded with many sessions
- Lock file uses timestamp + UUID but cleanup relies on isStale() check which runs on every acquire attempt
- `syncAllTasksToTodos` in todo-sync.ts has complex filtering logic that could miss edge cases with id-less todos

---

## Feature 8: Category-Based Delegation

### Overview

Categories are agent configuration presets that optimize model selection for specific task domains. Instead of specifying a model directly, delegations use a category name and the system resolves the appropriate model.

### Core Components

**1. Category Configuration**

**Default Categories (src/tools/delegate-task/constants.ts):**
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

**Category Prompt Appends** add domain-specific instructions:
- `visual-engineering`: Mandates design system analysis before coding, phase-based workflow
- `ultrabrain`: Deep reasoning, architectural decisions, strategic advisor mindset
- `deep`: Autonomous problem-solving, goal-oriented, minimal check-ins
- `quick`: Efficient execution, smaller model (gpt-5.4-mini), requires exhaustive explicitness
- `writing`: Anti-AI-slop rules (no em-dashes, no filler phrases, plain words)

**2. Category Resolution**

**src/tools/delegate-task/category-resolver.ts:**
- `resolveCategoryExecution()` handles the full resolution pipeline
- Model priority: explicit category model > sisyphus-junior override > category default > system default
- Handles model requirements (some categories require specific models)
- Builds fallback chains for retry on model errors
- Detects unstable agents (Gemini, MiniMax, Kimi) for special handling

**3. Session Category Registry**

**src/shared/session-category-registry.ts:**
```typescript
SessionCategoryRegistry {
  register(sessionID, category)   // Map session to category
  get(sessionID)                  // Get category for session
  remove(sessionID)               // Cleanup
  has(sessionID)                  // Check if registered
}
```
Used by runtime-fallback hook to lookup category-specific fallback_models.

**4. Dynamic Agent Prompt Builder**

**src/agents/dynamic-agent-prompt-builder.ts:**
- `buildCategorySkillsDelegationGuide()` - Generates the mandatory category+skill selection protocol
- Enforces VISUAL WORK = ALWAYS `visual-engineering` with zero tolerance
- Category domain matching table with clear mappings:
  - UI, styling, animations -> visual-engineering
  - Hard logic, architecture -> ultrabrain
  - Autonomous research + end-to-end -> deep
  - Single-file typo, trivial -> quick

**5. Sisyphus-Junior Agent**

**src/agents/sisyphus-junior/index.ts:**
- Route-based prompt builder for different model families:
  - GPT models -> gpt.ts / gpt-5-4.ts / gpt-5-3-codex.ts
  - Gemini models -> gemini.ts
  - Default (Claude) -> default.ts

**Blocked tools:** `["task"]` - Sisyphus-Junior cannot spawn other agents (prevents infinite recursion)

**Tool restrictions:** call_omo_agent IS allowed so subagents can spawn explore/librarian

**Thinking configuration:**
- GPT models: `reasoningEffort: "medium"`
- Non-GPT: `thinking: { type: "enabled", budgetTokens: 32000 }`

### Category-Skill Reminder Hook

**Purpose:** Reminds orchestrator agents (sisyphus, sisyphus-junior, atlas) about category+skill system when doing delegatable work.

**Trigger conditions:**
- Agent is in TARGET_AGENTS set
- Tool used is in DELEGATABLE_WORK_TOOLS (edit, write, bash, read, grep, glob)
- After 3+ tool calls WITHOUT using delegation tools (task, call_omo_agent)

**Implementation:** `src/hooks/category-skill-reminder/hook.ts`
- Tracks per-session state: delegationUsed, reminderShown, toolCallCount
- On session.deleted or session.compacted: clears session state
- Builds reminder message listing built-in vs custom skills

### Delegation Flow

When `task(category="visual-engineering", load_skills=["frontend-ui-ux"])` is called:

1. `resolveCategoryExecution()` resolves model config from category
2. `buildSisyphusJuniorPrompt()` selects prompt based on resolved model
3. `createSisyphusJuniorAgentWithOverrides()` creates agent config with:
   - Blocked tools: ["task"]
   - Allowed tools: everything except "task", plus "call_omo_agent"
   - Category prompt append injected
4. Skill content loaded via skill tool, merged into prompt
5. Agent spawned with category-optimized model

### Clever Solutions

**Domain matching enforcement:**
- Visual work -> visual-engineering has "ZERO TOLERANCE" enforcement
- Built into dynamic-agent-prompt-builder with explicit anti-patterns

**Model fallback chains:**
- Categories can specify fallback_models for retry scenarios
- Different from global fallback - category-specific

**Sisyphus-Junior routing:**
- Model-family-specific prompts allow optimization per model type
- GPT-specific handling for reasoningEffort vs thinking budgets

### Technical Debt / Edge Cases

**Custom category override gap:**
- If a user configures `categories.quick.model = "anthropic/claude-sonnet-4-6"`, the `QUICK_CATEGORY_PROMPT_APPEND` warning about gpt-5.4-mini becomes misleading
- Model selection in `resolveCategoryExecution()` handles this but prompt appends remain static

**Category validation:**
- Unknown categories produce error with list of available categories
- But disabled categories (via `disable: true`) silently return null without clear indication

---

## Feature 9: Built-in Skills

### Overview

Skills are pre-packaged workflow templates with optional embedded MCP servers. They provide specialized knowledge and step-by-step guidance for specific domains.

### Skill Structure

**Skill Definition (from .opencode/skills/github-triage/SKILL.md):**
```markdown
---
name: github-triage
description: "Read-only GitHub triage for issues AND PRs..."
---

# GitHub Triage - Read-Only Analyzer

<role>
Read-only GitHub triage orchestrator...
</role>

<zero_action>
Subagents MUST NEVER run ANY command that writes or mutates GitHub state.
</zero_action>

## Phase 1: Fetch All Open Items
...
```

**Frontmatter fields:**
- name, description (required)
- model (optional, overrides category model)
- agent (optional, restricts to specific agent)
- subtask (optional)
- license, compatibility, metadata
- allowed-tools (optional, restricts available tools)
- mcp (optional, embedded MCP config)

### Skill Loading

**src/features/opencode-skill-loader/** (35+ files):
- `loader.ts` - Main skill discovery and loading
- `skill-discovery.ts` - Finds skills in standard locations
- `skill-directory-loader.ts` - Loads skill from directory
- `loaded-skill-from-path.ts` - Converts raw files to LoadedSkill
- `skill-content.ts` - Extracts <skill-instruction> content

**Skill Scopes (priority order high to low):**
```typescript
scopePriority = {
  project: 4,           // .opencode/skills/ in project
  user: 3,              // ~/.config/opencode/skills/
  opencode: 2,          // Built-in plugin skills
  "opencode-project": 2,
  plugin: 1,            // From plugins
  config: 1,            // From config file
  builtin: 1,
}
```

**Skill discovery paths:**
- Project: `{cwd}/.opencode/skills/`
- User: `~/.config/opencode/skills/`
- Plugin: Built-in skills from plugin package

### Skill Tool

**src/tools/skill/tools.ts:**

**createSkillTool(options):**
- Caches description after first build
- Skills and commands discovered at startup or lazily
- Merges pre-provided skills with discovered ones
- Skill matching: exact match, case-insensitive
- Command matching: sorted by priority, exact match

**execute(args, ctx):**
1. Find skill or command by name (case-insensitive, slash optional)
2. Check agent restriction if specified
3. Extract skill body from `<skill-instruction>` tags
4. For git-master: inject GitMasterConfig
5. If skill has mcpConfig: format MCP capabilities
6. Return formatted skill content

**Skill Body Extraction:**
```typescript
extractSkillBody(skill):
  if skill.lazyContent:
    load() then extract <skill-instruction>...</skill-instruction>
  else if skill.path:
    extractSkillTemplate(skill)  // From file
  else:
    extract from skill.definition.template
```

### MCP Integration

**Skill-embedded MCPs:**
- Skills can specify `mcp` config for on-demand MCP servers
- Servers spin up scoped to task, disappear when done
- Keeps context window clean

**src/features/skill-mcp-manager/**:
- `listTools(info, context)` - Discover MCP tools
- `listResources(info, context)` - Discover resources
- `listPrompts(info, context)` - Discover prompts
- Tool output shows available MCP servers with usage instructions

### Git Master Skill

**Special handling in skill tool:**
```typescript
if (matchedSkill.name === "git-master") {
  body = injectGitMasterConfig(body, options.gitMasterConfig)
}
```

Allows configuration of watermark/co-author settings via GitMasterConfig.

### Available Skills (in repo)

**github-triage:**
- Read-only GitHub triage orchestrator
- 1 issue/PR = 1 background quick subagent
- Zero-action policy (no mutations)
- Evidence-backed reports to /tmp/

**work-with-pr:**
- Working with pull requests

**pre-publish-review:**
- Pre-publish review

**work-with-pr-workspace:**
- PR workspace support

### Skill vs Command Resolution

**Priority:**
1. Skills checked first (exact match, case-insensitive)
2. Commands checked second (sorted by scope priority)

**Error handling:**
- Partial match: suggest similar names
- No match: list all available skills/commands

### Clever Solutions

**Lazy content loading:**
- Skills use `lazyContent.load()` to defer loading until needed
- Prevents loading all skill content at startup

**Scope priority system:**
- Higher priority scopes override lower ones
- Project > User > Opencode > Plugin
- Clear precedence for skill overrides

**Skill-embedded MCP:**
- MCP servers only spin up when skill is invoked
- Scoped to task, cleaned up after
- Context-efficient vs always-on MCPs

### Technical Debt / Edge Cases

**Cache invalidation:**
- `clearSkillCache()` called before `getAllSkills()` in skill tool
- But concurrent calls could still get stale cache

**Agent restrictions:**
- If skill specifies `agent: "sisyphus"` but ctx.agent is different, throws error
- No fallback or warning - hard failure

**MCP server errors:**
- MCP capability formatting catches errors per-server
- Individual server failures don't block other servers
- But failed servers show "*Failed to connect*" message

**Circular skill dependencies:**
- No detection of circular skill includes
- Could cause infinite loops in template resolution
