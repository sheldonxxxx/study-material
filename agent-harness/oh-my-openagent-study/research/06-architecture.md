# oh-my-openagent Architecture Analysis

## Overview

**Project:** oh-my-opencode (v3.11.0)
**Type:** Monorepo Node.js/TypeScript plugin system
**Runtime:** Bun
**Core Architecture:** Hook-based Plugin System with Event-Driven Communication

The project is an AI Agent Harness that integrates with OpenCode as a plugin, providing multi-model orchestration, parallel background agents, LSP/AST tools, and extensive customization through a hook system.

---

## Architectural Pattern

### Primary Pattern: Hook-Based Event-Driven Architecture

The architecture is **not** traditional MVC, microservices, or pure layered architecture. Instead, it implements a **hook-based plugin architecture** where:

1. **Plugin Entry Point** (`src/index.ts`) acts as the composition root
2. **Hooks** are discrete, composable feature modules that intercept and transform OpenCode events
3. **Managers** provide stateful services (background tasks, tmux sessions, MCP connections)
4. **Event Bus** via OpenCode's plugin interface dispatches events to registered hooks
5. **Tools** are exposed to OpenCode via the plugin interface

### Communication Patterns

| Pattern | Usage |
|---------|-------|
| **Event-driven** | OpenCode events flow through `createEventHandler` which dispatches to all registered hooks |
| **Direct method calls** | Managers expose APIs (e.g., `backgroundManager.launch()`) |
| **Promise chains** | Async operations use Promise-based callbacks |
| **Shared state** | Module-level Maps for cross-hook state (session-agent registry, model cache) |
| **Dependency injection** | Factories receive `ctx`, `pluginConfig` and create instances |

---

## Module Structure and Responsibilities

### Core Entry Point (`src/`)

| File | Responsibility |
|------|----------------|
| `index.ts` | OhMyOpenCodePlugin - Main plugin factory. Orchestrates creation of managers, hooks, tools, and interface |
| `plugin-interface.ts` | `createPluginInterface()` - Assembles the PluginInterface returned to OpenCode |
| `create-hooks.ts` | `createHooks()` - Factory for composing all hooks (core, continuation, skill) |
| `create-managers.ts` | `createManagers()` - Factory for stateful services (BackgroundManager, TmuxSessionManager, SkillMcpManager) |
| `create-tools.ts` | `createTools()` - Factory for tool definitions exposed to OpenCode |
| `plugin-config.ts` | `loadPluginConfig()` - Loads and validates user configuration |
| `plugin-state.ts` | `createModelCacheState()` - Creates shared state containers |
| `plugin-dispose.ts` | `createPluginDispose()` - Cleanup function registry |

### Plugin Interface (`src/plugin/`)

The plugin interface is composed of **handlers** that receive OpenCode lifecycle events:

```
PluginInterface
|-- tool: ToolsRecord                    # Tool definitions
|-- chat.params                         # Transform chat parameters
|-- chat.headers                        # Modify request headers
|-- chat.message                        # Handle/transform chat messages
|-- experimental.chat.messages.transform # Transform message stream
|-- experimental.chat.system.transform   # Transform system prompt
|-- config                              # Configuration handler
|-- event                               # Central event dispatcher
|-- tool.execute.before                 # Pre-tool execution hooks
|-- tool.execute.after                  # Post-tool execution hooks
```

### Managers (`src/features/`)

| Manager | Purpose |
|---------|---------|
| `BackgroundManager` | Manages parallel background agent sessions with concurrency control, polling, and notification |
| `TmuxSessionManager` | Coordinates tmux integration for subagent panes |
| `SkillMcpManager` | Manages MCP server connections for skills |

### Hooks System (`src/hooks/`, `src/plugin/hooks/`)

Hooks are organized into three categories:

**1. Core Hooks** (`create-core-hooks.ts`):
- Session hooks (25+): context-window-monitor, session-recovery, preemptive-compaction, model-fallback, etc.
- Tool guard hooks: tool-restriction enforcement
- Transform hooks: message/system transformations

**2. Continuation Hooks** (`create-continuation-hooks.ts`):
- Claude Code hooks integration
- Auto-update checker
- Background notification
- Todo continuation enforcer

**3. Skill Hooks** (`create-skill-hooks.ts`):
- Skill loading and context injection
- Skill MCP manager integration
- Auto slash command detection

---

## Key Abstractions and Interfaces

### Plugin Context

```typescript
type PluginContext = Parameters<Plugin>[0]
// Contains: { directory: string, client: PluginClient }
// PluginClient provides: session CRUD, prompts, tui, tools
```

### Hook Contract

Hooks implement a common contract with optional lifecycle methods:

```typescript
type Hook = {
  event?: (input: EventInput) => Promise<void>
  "tool.execute.before"?: (input, output) => Promise<void>
  "tool.execute.after"?: (input, output) => Promise<void>
  "chat.message"?: (input, output) => Promise<void>
  dispose?: () => void
}
```

### Agent Factory Pattern

```typescript
type AgentFactory = ((model: string) => AgentConfig) & {
  mode: AgentMode  // "primary" | "subagent" | "all"
}
```

Agents are parameterized factories that receive a model name and return an `AgentConfig`.

### Tool Definition

Tools are defined via OpenCode's `ToolDefinition` interface and registered in the plugin's `tool` export.

---

## Design Patterns in Use

### 1. Factory Pattern
- `createHooks()`, `createManagers()`, `createTools()` are factory functions
- Each hook type has a `createXxxHook()` factory

### 2. Singleton Pattern (with caveats)
- Managers are created once per plugin instance
- `lspManager`, `getTaskToastManager()` use module-level singletons
- BackgroundManager is passed around but has internal singleton-like state

### 3. Observer/Event Subscription Pattern
- `createEventHandler` dispatches to all registered hooks
- Hooks subscribe to specific event types

### 4. Strategy Pattern
- Multiple alternative implementations selectable at runtime (e.g., model fallback strategies, retry strategies)
- `RecoveryStrategy`, `FallbackModelObject` types support pluggable strategies

### 5. State Machine Pattern
- Background tasks follow a clear state machine: `pending` -> `running` -> `completed/error/cancelled`
- Session status classification (`isActiveSessionStatus`, `isTerminalSessionStatus`)

### 6. Template Method Pattern
- Hook creation uses common patterns via `safeCreateHook()` wrapper
- `createCoreHooks`, `createContinuationHooks`, `createSkillHooks` share structure

### 7. Proxy/Middleware Pattern
- Transform hooks (`messages-transform.ts`, `system-transform.ts`) wrap and modify data passing through
- `tool-execute-before` and `tool-execute-after` act as middleware

### 8. Registry Pattern
- `SessionCategoryRegistry`, `ToolMetadataStore`, `ConnectedProvidersCache`
- Global registries for cross-cutting concerns

### 9. Builder/Fluent Pattern
- `buildAgent()` in `agent-builder.ts` chains configuration
- `PromptSectionBuilder` in Atlas agent

### 10. Command Pattern
- CLI commands in `src/cli/` follow command pattern
- Each command (install, doctor, run) is a separate module

---

## Major Modules and Their Responsibilities

### Agent System (`src/agents/`)

| Agent | Role |
|-------|------|
| **Sisyphus** | Primary planning/coding agent |
| **Hephaestus** | Code generation specialist (GPT family) |
| **Atlas** | Architecture/planning agent with wave-based approval |
| **Sisyphus-Junior** | Lighter planning agent |
| **Prometheus** | High-accuracy mode agent |
| **Oracle** | Exploration/diagnostic agent |
| **Librarian** | External library research |
| **Momus** | Context management |
| **Metis** | Knowledge/reasoning |
| **Explore** | File/code exploration |

Each agent has:
- Base implementation (`src/agents/{name}/`)
- Prompt variants for different providers (`gpt.ts`, `gemini.ts`)
- Skill integration via `builtin-agents/`

### Background Agent (`src/features/background-agent/`)

Complex stateful service managing:
- Task lifecycle (launch, track, resume, complete, cancel)
- Concurrency control per model/agent
- Session polling and idle detection
- Fallback retry logic
- Parent notification injection
- Circuit breaker for infinite loops
- Subagent depth/spawn limits

### Tool System (`src/tools/`)

Tools implemented as separate modules:
- `glob/`, `grep/`, `ast-grep/` - Search tools
- `lsp/` - Language Server Protocol integration
- `hashline-edit/` - Edit tool with hashline support
- `interactive-bash/` - Bash execution
- `background-task/`, `delegate-task/` - Task delegation
- `call-omo-agent/` - Cross-agent communication
- `look-at/` - Vision tool

### Configuration Schema (`src/config/schema/`)

Zod-based schema validation for all configuration:
- `oh-my-opencode-config.ts` - Root config
- `agent-overrides.ts` - Per-agent customization
- `background-task.ts` - Task settings
- `hooks.ts` - Hook enable/disable
- `tmux.ts` - Tmux layout
- `runtime-fallback.ts` - Model fallback chain
- `categories.ts` - Agent categories

---

## Data Flow

### Plugin Initialization Flow

```
OhMyOpenCodePlugin(ctx)
  1. loadPluginConfig(ctx.directory)
  2. createModelCacheState()
  3. createManagers(ctx, pluginConfig)
     - TmuxSessionManager
     - BackgroundManager
     - SkillMcpManager
     - ConfigHandler
  4. createTools(ctx, pluginConfig, managers)
  5. createHooks(ctx, pluginConfig, managers, tools)
     - createCoreHooks()
     - createContinuationHooks()
     - createSkillHooks()
  6. createPluginInterface(managers, hooks, tools)
  7. Return PluginInterface
```

### Event Processing Flow

```
OpenCode emits event
  -> createEventHandler (central dispatcher)
     -> dispatchToHooks() (sequential Promise chain)
        -> autoUpdateChecker.event()
        -> claudeCodeHooks.event()
        -> backgroundNotificationHook.event()
        -> sessionNotification()
        -> todoContinuationEnforcer.handler()
        -> ... (20+ more hooks)
        -> contextWindowMonitor.event()
        -> preemptiveCompaction.event()
        -> runtimeFallback.event()
        -> ... etc
     -> Synthetic idle normalization
     -> dispatchToHooks(syntheticIdle)
     -> Session lifecycle handling (created/deleted)
     -> Message updated handling (agent tracking, model fallback)
     -> Session status handling (retry -> fallback)
     -> Error handling (session.error -> recovery/fallback)
```

### Tool Execution Flow

```
Tool called by agent
  -> tool.execute.before handlers
     -> toolOutputTruncator
     -> claudeCodeHooks
     -> preemptiveCompaction
     -> ... (20+ more)
  -> Tool executes
  -> tool.execute.after handlers
     -> Same hook list in reverse-ish order
```

---

## Cross-Cutting Concerns

### State Management

| State Type | Storage |
|------------|---------|
| Session-agent mapping | `claude-code-session-state.ts` (Map) |
| Session model | `session-model-state.ts` (Map) |
| Session tools | `session-tools-store.ts` (Map) |
| Session prompt params | `session-prompt-params-state.ts` (Map) |
| Model cache | `plugin-state.ts` (ModelCacheState) |
| Tool metadata | `tool-metadata-store.ts` (Map) |
| Connected providers | `connected-providers-cache.ts` (Map) |

### Error Handling

- **Recovery**: `session-recovery.ts` handles thinking block errors
- **Fallback**: `runtime-fallback.ts` and `model-fallback.ts` handle API errors
- **Retry**: `retry-status-utils.ts`, `model-error-classifier.ts`
- **Circuit breaker**: Loop detection in background tasks

### Logging

`src/shared/logger.ts` provides structured logging with context:
```typescript
log("[component] message", { key: value })
```

---

## Extension Points

### Adding a New Hook

1. Create factory in `src/hooks/{hook-name}/`
2. Implement hook interface (event, tool.execute.before/after, etc.)
3. Add to appropriate `createXxxHooks()` factory
4. Register in `dispatchToHooks()` in `event.ts`
5. Add to `tool-execute-after.ts` if tool hooks needed

### Adding a New Tool

1. Create tool module in `src/tools/{tool-name}/`
2. Implement `ToolDefinition` interface
3. Add to `createTools()` in `create-tools.ts`
4. Register in plugin interface `tool` export

### Adding a New Agent

1. Create agent directory in `src/agents/{agent-name}/`
2. Implement agent factory with `mode` property
3. Register in `builtin-agents.ts`
4. Add to `AgentPromptMetadata` registry if using dynamic prompts

---

## Configuration Architecture

User configuration flows:

```
opencode.json / opencode.config.js
  -> loadPluginConfig() [plugin-config.ts]
  -> Zod validation [config/schema/]
  -> OhMyOpenCodeConfig type
  -> Passed to all factories (hooks, managers, tools)
  -> Hooks check isHookEnabled(name) before creating
```

Key config sections:
- `agents` - Per-agent model/prompt overrides
- `categories` - Agent grouping with default models
- `disabled_hooks` - Kill switches for specific hooks
- `background_task` - Concurrency limits, circuit breaker
- `runtime_fallback` - Error-based model switching
- `model_fallback` - Explicit fallback chains
- `tmux` - Layout configuration

---

## Key Files Reference

| Path | Purpose |
|------|---------|
| `src/index.ts` | Plugin entry point, composition root |
| `src/plugin/types.ts` | PluginInterface, ToolsRecord, TmuxConfig types |
| `src/plugin/event.ts` | Central event dispatcher |
| `src/plugin/tool-execute-after.ts` | Tool execution pipeline |
| `src/create-hooks.ts` | Hook composition factory |
| `src/create-managers.ts` | Manager composition factory |
| `src/features/background-agent/manager.ts` | Background task orchestration |
| `src/agents/types.ts` | AgentFactory, AgentMode, AgentPromptMetadata |
| `src/config/schema/oh-my-opencode-config.ts` | Root config schema |
| `src/shared/` | Utilities, logger, state registries |
