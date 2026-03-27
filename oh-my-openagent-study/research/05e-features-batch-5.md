# Features Batch 5 Research (Features 13-15)

**Date:** 2026-03-26
**Features:** Session Recovery & Model Fallback, Claude Code Compatibility Layer, Session Notifications

---

## Feature 13: Session Recovery & Model Fallback

### Overview

Multi-layer recovery system with three distinct mechanisms:
1. **Session Recovery** (`session-recovery` hook) - Handles missing tool results and thinking block issues
2. **Runtime Fallback** (`runtime-fallback` hook) - Automatic model switching on retryable API errors
3. **Model Fallback** (`model-fallback` hook) - Fallback chain management when primary models unavailable

### Core Implementation Files

| File | Purpose |
|------|---------|
| `src/hooks/session-recovery/hook.ts` | Main session recovery orchestrator |
| `src/hooks/session-recovery/detect-error-type.ts` | Error classification for recovery |
| `src/hooks/session-recovery/recover-tool-result-missing.ts` | Tool crash recovery |
| `src/hooks/runtime-fallback/hook.ts` | Runtime fallback orchestrator |
| `src/hooks/runtime-fallback/error-classifier.ts` | Error classification for fallback |
| `src/hooks/runtime-fallback/event-handler.ts` | Event handling for fallback |
| `src/hooks/runtime-fallback/auto-retry.ts` | Auto-retry logic |
| `src/hooks/model-fallback/hook.ts` | Model availability fallback |
| `src/shared/model-error-classifier.ts` | Shared error classification |
| `src/shared/model-requirements.ts` | Fallback chain definitions |
| `src/shared/fallback-chain-from-models.ts` | Fallback chain parsing |

### Session Recovery Flow

```
Error occurs in session
         |
         v
detectErrorType(error) --> RecoveryErrorType
         |
         v
handleSessionRecovery(info)
         |
    +----+----+
    |         |
    v         v
Abort session  Get failed message
    |         |
    v         v
Toast shown   Pattern match on error type
    |         |
    v         v
Call recovery function (recoverToolResultMissing, etc.)
    |
    v
If auto_resume enabled, call resumeSession()
```

**Error Types Detected:**
- `tool_result_missing` - Tool use without corresponding result
- `unavailable_tool` - Model called a tool that doesn't exist
- `thinking_block_order` - Thinking blocks in wrong order
- `thinking_disabled_violation` - Thinking block used when disabled
- `assistant_prefill_unsupported` - Prefill not supported (non-recoverable)

**Key Recovery Mechanism - `recoverToolResultMissing`:**
- Extracts tool_use IDs from the failed message
- Injects "Operation cancelled by user (ESC pressed)" tool results
- Uses `session.promptAsync` to deliver the recovery
- Handles both SQLite and SDK fallback storage backends

**Key Recovery Mechanism - `recoverThinkingBlockOrder`:**
- Attempts to fix thinking block ordering issues
- If `experimental.auto_resume` is enabled, auto-resumes the session after recovery

**Key Recovery Mechanism - `recoverThinkingDisabledViolation`:**
- Strips thinking blocks when they're not allowed
- Similar auto-resume behavior as thinking block order recovery

### Runtime Fallback Flow

```
session.error event
         |
         v
isRetryableError() check
         |
    +----+----+
    |         |
   Yes        No
    |         |
    v         v
Extract status code  Skip
    |         |
    v         v
getFallbackModelsForSession()
    |
    v
prepareFallback() --> next model in chain
    |
    v
scheduleSessionFallbackTimeout()
    |
    v
promptAsync() with new model + retry parts
```

**Retryable Errors (from `error-classifier.ts`):**
- Status codes: 429, 503, 529
- Error names: `providermodelnotfounderror`, `ratelimiterror`, `quotaexceedederror`, `insufficientcreditserror`, `modelunavailableerror`, `providerconnectionerror`, `authenticationerror`, `freeusagelimiterror`

**Non-Retryable Errors:**
- `messageabortederror`, `permissiondeniederror`, `contextlengtherror`, `timeouterror`, `validationerror`, `syntaxerror`, `usererror`

**Session State Management:**
- Uses Maps to track per-session state: `sessionStates`, `sessionLastAccess`, `sessionRetryInFlight`, `sessionAwaitingFallbackResult`
- Cleanup interval runs every 5 minutes to remove stale sessions (30-min TTL)
- Timeout mechanism triggers fallback retry if session hangs

### Model Fallback Flow

```
setPendingModelFallback(sessionID, agent, provider, model)
         |
         v
Look up AGENT_MODEL_REQUIREMENTS[agentKey].fallbackChain
         |
         v
Store pending fallback in module-level Map
         |
         v
chat.message handler calls getNextFallback()
         |
    +----+----+
    |         |
   Yes        No
    |         |
    v         v
Get next fallback  Return null
from chain         |
    |         |
    v         v
Apply to output    No change
message.model
```

**Fallback Chain Structure (from `model-requirements.ts`):**
```typescript
type FallbackEntry = {
  providers: string[];      // Preferred providers in order
  model: string;            // Model ID
  variant?: string;         // e.g., "max", "medium", "high"
  reasoningEffort?: string;
  temperature?: number;
  top_p?: number;
  maxTokens?: number;
  thinking?: { type: "enabled" | "disabled"; budgetTokens?: number };
};
```

**Sisyphus Fallback Chain Example:**
1. `anthropic/github-copilot/opencode` + `claude-opus-4-6` (variant: max)
2. `opencode-go` + `kimi-k2.5`
3. `kimi-for-coding` + `k2p5`
4. Multiple providers + `kimi-k2.5`
5. `openai/github-copilot/opencode` + `gpt-5.4` (variant: medium)
6. `zai-coding-plan/opencode` + `glm-5`
7. `opencode` + `big-pickle`

**Key Link - Provider Connectivity Check:**
```typescript
const isReachable = (entry: FallbackEntry): boolean => {
  if (!connectedSet) return true;
  // Gate only on provider connectivity, not model list
  if (entry.providers.some(p => connectedSet.has(p.toLowerCase()))) {
    return true;
  }
  const preferredProvider = state.providerID.toLowerCase();
  return connectedSet.has(preferredProvider);
};
```

### Technical Decisions & Shortcuts

1. **No-op Fallback Skip:** Skips fallbacks where provider/model are identical to current
2. **Provider-First Model Lists:** Model lists can be stale (users manually add models), so only provider connectivity is checked for reachability
3. **Attempt Count Preservation:** When retrying in same session, `attemptCount` is preserved across `session.error` retries
4. **Module-Level State:** `pendingModelFallbacks`, `lastToastKey`, `sessionFallbackChains` are module-level Maps (singleton pattern)
5. **Fallback Bootstrap Model Resolution:** If state doesn't exist when error occurs, attempts to resolve model from session context

### Edge Cases

1. **Exhausted Fallback Chain:** Returns `false` from `setPendingModelFallback` when all fallbacks used
2. **Retry Already In Flight:** Skips if `sessionRetryInFlight` already contains session ID
3. **Session Timeout While Retrying:** Clears retry state and triggers new fallback
4. **SQLite vs SDK Storage:** `recoverToolResultMissing` handles both storage backends
5. **Activity During Idle Notification:** `activityGracePeriodMs` ignores late-arriving activity events

---

## Feature 14: Claude Code Compatibility Layer

### Overview

Full compatibility for Claude Code hooks, commands, skills, agents, MCPs, and plugins. Supports loading from `~/.claude/`, `.claude/`, and Claude Code's `settings.json` hooks system.

### Core Implementation Files

| File | Purpose |
|------|---------|
| `src/plugin-handlers/config-handler.ts` | Main config application orchestrator |
| `src/plugin-handlers/agent-config-handler.ts` | Agent loading and merging |
| `src/plugin-handlers/mcp-config-handler.ts` | MCP server loading |
| `src/plugin-handlers/command-config-handler.ts` | Command/skill loading |
| `src/plugin-handlers/tool-config-handler.ts` | Tool configuration |
| `src/plugin-handlers/provider-config-handler.ts` | Provider configuration |
| `src/hooks/claude-code-hooks/claude-code-hooks-hook.ts` | Hook system bridge |
| `src/hooks/claude-code-hooks/handlers/session-event-handler.ts` | Session event handling |
| `src/features/claude-code-agent-loader.ts` | Load agents from Claude Code paths |
| `src/features/claude-code-mcp-loader.ts` | Load MCP from Claude Code paths |
| `src/features/claude-code-command-loader.ts` | Load commands from Claude Code paths |
| `src/features/opencode-skill-loader.ts` | Skill discovery and loading |

### Configuration Flow

```
createConfigHandler()
         |
         v
applyProviderConfig() --> Configure providers
         |
         v
loadPluginComponents() --> Load builtin plugin components
         |
         v
applyAgentConfig() --> Merge agents from all sources
         |
         v
applyToolConfig() --> Configure tools
         |
         v
applyMcpConfig() --> Merge MCP servers
         |
         v
applyCommandConfig() --> Merge commands and skills
```

### Agent Loading Priority (from `agent-config-handler.ts`)

1. **Builtin agents** (from `createBuiltinAgents`)
2. **Sisyphus configuration** (if Sisyphus enabled):
   - Sisyphus as main orchestrator
   - Sisyphus-Junior as executor
   - OpenCode-Builder (if `builderEnabled`)
   - Prometheus planner (if `plannerEnabled`)
3. **Filtered user agents** (from `~/.claude/`)
4. **Filtered project agents** (from `.claude/`)
5. **Filtered plugin agents** (from plugin components)
6. **Custom config agents** (from opencode.json)

### Agent Override Protection

```typescript
const createProtectedAgentNameSet = (names: string[]) => new Set(names.map(n => n.toLowerCase()));

const filterProtectedAgentOverrides = (agents, protectedNames) => {
  // Removes any user/project agents that would override built-in agents
  return Object.fromEntries(
    Object.entries(agents).filter(([name]) => !protectedNames.has(name.toLowerCase()))
  );
};
```

**Protected Builtin Agents:** sisyphus, sisyphus-junior, prometheus, OpenCode-Builder, atlas, and all builtin agents

### Skill Discovery Order (from `applyAgentConfig`)

1. Config source skills (from `pluginConfig.skills`)
2. OpenCode project skills
3. Project Claude skills (from `.claude/skills`)
4. Project agents skills (from `.claude/agents`)
5. OpenCode global skills
6. User Claude skills (from `~/.claude/skills`)
7. Global agents skills (from `~/.claude/agents`)

### Claude Code Hooks Bridge

```typescript
createClaudeCodeHooksHook(ctx, config, contextCollector)
         |
    +----+----+----+
    |    |    |    |
    v    v    v    v
pre-compact chat.message tool.execute.before tool.execute.after session.event
```

**Session Event Handler:**
- Handles `session.error` --> Stores error state
- Handles `session.deleted` --> Clears hook state
- Handles `session.idle` --> Executes Stop hooks with Claude Code config

### MCP Configuration Merge Order

```typescript
{
  ...createBuiltinMcps(disabledMcps),      // Built-in MCPs first
  ...userMcp,                               // User config MCPs
  ...mcpResult.servers,                      // Claude Code MCPs
  ...pluginComponents.mcpServers,           // Plugin MCPs
}
```

**User-disabled MCPs** are preserved even if Claude Code would enable them.

### Command Configuration Merge Order

```typescript
{
  ...builtinCommands,
  ...skillsToCommandDefinitionRecord(configSourceSkills),
  ...userCommands,
  ...userSkills,
  ...globalAgentsSkills,
  ...opencodeGlobalCommands,
  ...opencodeGlobalSkills,
  ...systemCommands,
  ...projectCommands,
  ...projectSkills,
  ...projectAgentsSkills,
  ...opencodeProjectCommands,
  ...opencodeProjectSkills,
  ...pluginComponents.commands,
  ...pluginComponents.skills,
}
```

### Technical Decisions & Shortcuts

1. **Parallel Discovery:** All skill/agent/command discovery runs in `Promise.all()` for speed
2. **Agent Key Remapping:** Maps Claude Code agent names to Oh-My-OpenCode display names via `AGENT_NAME_MAP`
3. **Priority Ordering:** `reorderAgentsByPriority()` ensures proper agent ordering after merge
4. **Prometheus Config Builder:** Special handling to build Prometheus agent with proper overrides
5. **Plan Demotion:** When Prometheus is enabled and `replace_plan` is true, plan agent is demoted

### Edge Cases

1. **Parent Session Skip:** Auto-update-checker ignores child sessions (`parentID` exists)
2. **Disabled Agents:** Properly migrated via `AGENT_NAME_MAP` before filtering
3. **Sisyphus Disabled:** Falls back to standard agent loading without Sisyphus orchestration
4. **Skill-Embedded MCPs:** Skills can carry their own MCP servers via `skill-mcp-manager`

---

## Feature 15: Session Notifications & Auto-Update Checking

### Overview

OS-native notifications when background agents complete or agents go idle. Works on macOS, Linux, and Windows. Auto-update-checker notifies of new versions on session creation.

### Core Implementation Files

| File | Purpose |
|------|---------|
| `src/hooks/session-notification.ts` | Main session notification hook |
| `src/hooks/session-notification-sender.ts` | Platform-specific notification sending |
| `src/hooks/session-notification-scheduler.ts` | Idle notification scheduling |
| `src/hooks/session-notification-utils.ts` | Utility functions |
| `src/hooks/background-notification/hook.ts` | Background agent notification routing |
| `src/hooks/auto-update-checker/hook.ts` | Auto-update check hook |

### Session Notification Architecture

```
createSessionNotification(ctx, config)
         |
         v
detectPlatform() --> darwin | linux | win32 | unsupported
         |
         v
createIdleNotificationScheduler()
         |
         v
Event handlers:
  - session.created --> markSessionActivity()
  - session.idle --> scheduleIdleNotification()
  - message.updated --> markSessionActivity()
  - tool.execute.before/after --> markSessionActivity()
  - permission.* --> Send permission notification
  - session.deleted --> deleteSession()
```

### Platform-Specific Notification Sending

**macOS:**
1. Try `terminal-notifier` first (deterministic click-to-focus)
2. Fallback to `osascript` (may open Finder on click)

**Linux:**
- Uses `notify-send` command

**Windows:**
- Uses PowerShell toast script via `BurntToast` or similar

### Idle Notification Scheduler

```typescript
createIdleNotificationScheduler({
  ctx,
  platform,
  config: {
    idleConfirmationDelay: 1500,  // Wait before notifying
    skipIfIncompleteTodos: true,  // Skip if todos pending
    maxTrackedSessions: 100,     // Session limit before cleanup
    enforceMainSessionFilter: true,  // Only main session
    activityGracePeriodMs: 100,     // Ignore late activity
  },
  hasIncompleteTodos,
  send,  // Actual notification send function
  playSound,
})
```

**Scheduler Logic:**
1. On `session.idle`, schedule notification after `idleConfirmationDelay`
2. If activity occurs before delay, cancel scheduled notification
3. `activityGracePeriodMs` ignores late-arriving activity events
4. Notifications are filtered by session type (subagent sessions filtered out)

### Notification Types

1. **Idle Notification:** "Agent is ready for input"
2. **Question Notification:** "Agent is asking a question" (when `question` tool called)
3. **Permission Notification:** "Agent needs permission to continue" (for permission events or permission-related question text)

### Background Notification Hook

```typescript
createBackgroundNotificationHook(manager: BackgroundManager)
         |
    +----+----+
    |         |
    v         v
event handler   chat.message handler
    |                 |
    v                 v
manager.handleEvent()  manager.injectPendingNotificationsIntoChatMessage()
```

**Key Insight:** Notifications are now delivered directly via `session.prompt({ noReply })` from the BackgroundManager, so the hook only handles event routing.

### Auto-Update Checker

```typescript
createAutoUpdateCheckerHook(ctx, options)
         |
         v
On first session.created (non-child session):
         |
         v
getCachedVersion() --> Check npm/package version
getLocalDevVersion() --> Check if running from source
         |
         v
showConfigErrorsIfAny()
updateAndShowConnectedProvidersCacheStatus()
refreshModelCapabilitiesOnStartup()
showModelCacheWarningIfNeeded()
         |
    +----+----+
    |         |
  Local     Run background update check
  dev         |
  mode        v
    |    showVersionToast() --> "OpenCode is now on Steroids..."
    |         |
    v         v
showLocalDevToast()  runBackgroundUpdateCheck()
```

### Clever Solutions

1. **Grace Period for Activity:** `activityGracePeriodMs` prevents notification cancellation from race conditions with late-arriving events
2. **Platform Detection Caching:** `currentPlatform` cached at creation time, not per-event
3. **Sound Path Per Platform:** Default sounds differ by OS (`/System/Library/Sounds/Glass.aiff` on macOS, etc.)
4. **Subagent Filtering:** Background agent sessions filtered out via `subagentSessions` set
5. **Main Session Enforcement:** Optional filter ensures only main session generates notifications

### Technical Decisions & Shortcuts

1. **Notification Cancellation via Activity:** Activity marking cancels scheduled idle notifications
2. **Tool Name Normalization:** Question tools checked case-insensitively
3. **Permission Event Set:** `permission.ask`, `permission.asked`, `permission.updated`, `permission.requested` all trigger permission notification
4. **Question Text Analysis:** Uses regex to detect permission-related keywords in question text

### Edge Cases

1. **Unsupported Platform:** Early return if `detectPlatform()` returns "unsupported"
2. **Missing Session ID:** All handlers gracefully skip if sessionID cannot be extracted
3. **Missing Client Methods:** Falls back to simple notification without content inspection
4. **Child Sessions:** Auto-update checker skips sessions with `parentID` (background/child sessions)
5. **Incomplete Todos:** Notifications skipped if `hasIncompleteTodos()` returns true and `skipIfIncompleteTodos` is set

### Notification Content Building

```typescript
buildReadyNotificationContent(hookCtx, {
  sessionID,
  baseTitle: "OpenCode",
  baseMessage: "Agent is ready for input",
})
         |
         v
Fetch session messages via hookCtx.client.session.messages()
         |
         v
Extract last assistant message text
         |
         v
Return { title, message } with context
```

---

## Cross-Feature Interactions

1. **Session Recovery + Model Fallback:** If session recovery fails and `experimental.auto_resume` is enabled, it triggers a new session prompt which could also invoke model fallback if configured

2. **Claude Code Compatibility + Session Notifications:** The Claude Code hooks system provides the bridge that allows session events (idle, error, etc.) to flow to the notification system

3. **Background Notifications + Session Notifications:** Background agent completion notifications flow through `BackgroundManager` which integrates with the session notification scheduler

4. **Runtime Fallback + Model Fallback:** Runtime fallback handles transient API errors with retry, while model fallback handles persistent unavailability through fallback chain progression

---

## Summary

These three features form a robust resilience and user experience layer:

- **Session Recovery** is defensive, catching specific error patterns and attempting surgical fixes
- **Runtime/Model Fallback** is proactive, switching models before failures impact user productivity
- **Claude Code Compatibility** is expansive, ensuring the ecosystem of Claude Code configs, skills, and agents all work seamlessly
- **Session Notifications** is informative, keeping users aware of agent state without constant monitoring
