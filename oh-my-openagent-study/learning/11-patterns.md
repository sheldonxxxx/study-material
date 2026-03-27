# oh-my-openagent Design Patterns

## Pattern Index

1. [Factory Pattern](#1-factory-pattern)
2. [Singleton Pattern](#2-singleton-pattern)
3. [Observer/Event Subscription Pattern](#3-observerevent-subscription-pattern)
4. [Strategy Pattern](#4-strategy-pattern)
5. [State Machine Pattern](#5-state-machine-pattern)
6. [Template Method Pattern](#6-template-method-pattern)
7. [Proxy/Middleware Pattern](#7-proxymiddleware-pattern)
8. [Registry Pattern](#8-registry-pattern)
9. [Builder/Fluent Pattern](#9-builderfluent-pattern)
10. [Command Pattern](#10-command-pattern)
11. [Dependency Injection](#11-dependency-injection)
12. [Composition Pattern](#12-composition-pattern)

---

## 1. Factory Pattern

**Purpose:** Create families of related objects without specifying concrete classes.

**Evidence:**

### Hook Factory (`src/create-hooks.ts`)
```typescript
export function createHooks(args: {
  ctx: PluginContext
  pluginConfig: OhMyOpenCodeConfig
  modelCacheState: ModelCacheState
  backgroundManager: BackgroundManager
  isHookEnabled: (hookName: HookName) => boolean
  safeHookEnabled: boolean
  mergedSkills: LoadedSkill[]
  availableSkills: AvailableSkill[]
}) {
  const core = createCoreHooks({ ctx, pluginConfig, modelCacheState, isHookEnabled, safeHookEnabled })
  const continuation = createContinuationHooks({ ctx, pluginConfig, isHookEnabled, safeHookEnabled, backgroundManager, sessionRecovery: core.sessionRecovery })
  const skill = createSkillHooks({ ctx, pluginConfig, isHookEnabled, safeHookEnabled, mergedSkills, availableSkills })

  return { ...core, ...continuation, ...skill }
}
```

### Manager Factory (`src/create-managers.ts`)
```typescript
export function createManagers(args: {
  ctx: PluginContext
  pluginConfig: OhMyOpenCodeConfig
  tmuxConfig: TmuxConfig
  modelCacheState: ModelCacheState
  backgroundNotificationHookEnabled: boolean
}): Managers {
  const tmuxSessionManager = new TmuxSessionManager(ctx, tmuxConfig)
  const backgroundManager = new BackgroundManager(ctx, pluginConfig.background_task, { ... })
  const skillMcpManager = new SkillMcpManager()
  const configHandler = createConfigHandler({ ctx, pluginConfig, modelCacheState })

  return { tmuxSessionManager, backgroundManager, skillMcpManager, configHandler }
}
```

### Agent Factory (`src/agents/types.ts`)
```typescript
export type AgentFactory = ((model: string) => AgentConfig) & {
  mode: AgentMode  // "primary" | "subagent" | "all"
}

export function createLibrarianAgent(model: string): AgentConfig {
  return {
    description: "Specialized codebase understanding agent...",
    mode: MODE,
    model,
    temperature: 0.1,
    ...restrictions,
    prompt: `# THE LIBRARIAN...`
  }
}
createLibrarianAgent.mode = MODE
```

---

## 2. Singleton Pattern

**Purpose:** Ensure a class has only one instance with global access point.

**Evidence:**

### Module-level singleton registries (`src/shared/session-category-registry.ts`)
```typescript
// Map of sessionID -> category name
const sessionCategoryMap = new Map<string, string>()

export const SessionCategoryRegistry = {
  register: (sessionID: string, category: string): void => {
    sessionCategoryMap.set(sessionID, category)
  },
  get: (sessionID: string): string | undefined => {
    return sessionCategoryMap.get(sessionID)
  },
  // ...
}
```

### Task Toast Manager (`src/features/task-toast-manager.ts`)
```typescript
let manager: TaskToastManager | null = null

export function getTaskToastManager(): TaskToastManager {
  if (!manager) {
    manager = new TaskToastManager()
  }
  return manager
}
```

### LSP Manager (`src/tools/lsp/index.ts`)
```typescript
export const lspManager = new LspManager()
```

---

## 3. Observer/Event Subscription Pattern

**Purpose:** Define a one-to-many dependency so when one object changes state, all dependents are notified.

**Evidence:**

### Central Event Dispatcher (`src/plugin/event.ts`)
```typescript
const dispatchToHooks = async (input: EventInput): Promise<void> => {
  await Promise.resolve(hooks.autoUpdateChecker?.event?.(input))
  await Promise.resolve(hooks.claudeCodeHooks?.event?.(input))
  await Promise.resolve(hooks.backgroundNotificationHook?.event?.(input))
  await Promise.resolve(hooks.sessionNotification?.(input))
  await Promise.resolve(hooks.todoContinuationEnforcer?.handler?.(input))
  await Promise.resolve(hooks.unstableAgentBabysitter?.event?.(input))
  await Promise.resolve(hooks.contextWindowMonitor?.event?.(input))
  await Promise.resolve(hooks.preemptiveCompaction?.event?.(input))
  // ... 20+ more hooks
}
```

### Hook Contract
```typescript
type Hook = {
  event?: (input: EventInput) => Promise<void>
  "tool.execute.before"?: (input, output) => Promise<void>
  "tool.execute.after"?: (input, output) => Promise<void>
  dispose?: () => void
}
```

---

## 4. Strategy Pattern

**Purpose:** Define a family of algorithms, encapsulate each one, and make them interchangeable.

**Evidence:**

### Fallback Chain Resolution (`src/shared/fallback-chain-from-models.ts`)
```typescript
export function findMostSpecificFallbackEntry(
  providerID: string,
  modelID: string,
  chain: FallbackEntry[],
): FallbackEntry | undefined {
  const resolved = `${providerID}/${modelID}`.toLowerCase()
  const matches: { entry: FallbackEntry; matchLen: number }[] = []

  for (const entry of chain) {
    for (const p of entry.providers) {
      const candidate = `${p}/${entry.model}`.toLowerCase()
      if (resolved.startsWith(candidate)) {
        matches.push({ entry, matchLen: candidate.length })
        break
      }
    }
  }

  if (matches.length === 0) return undefined
  matches.sort((a, b) => b.matchLen - a.matchLen)
  return matches[0].entry
}
```

### Model Fallback Config (`src/config/schema/fallback-models.ts`)
```typescript
export const FallbackModelObjectSchema = z.object({
  model: z.string(),
  variant: z.string().optional(),
  reasoningEffort: z.enum(["low", "medium", "high"]).optional(),
  temperature: z.number().optional(),
  top_p: z.number().optional(),
  maxTokens: z.number().optional(),
  thinking: z.boolean().optional(),
})
```

### Error Classification Strategy (`src/shared/model-error-classifier.ts`)
```typescript
export function shouldRetryError(error: { name?: string; message: string }): boolean {
  // Rate limit errors
  if (error.message.includes("429") || error.message.includes("rate_limit")) return true
  // Quota exceeded
  if (error.message.includes("quota") || error.message.includes("exceeded")) return true
  // Model not found - try fallback
  if (error.message.includes("model") && error.message.includes("not found")) return true
  // ...
}
```

---

## 5. State Machine Pattern

**Purpose:** Allow an object to alter its behavior when its internal state changes.

**Evidence:**

### Session Status Classification (`src/features/background-agent/session-status-classifier.ts`)
```typescript
export function isActiveSessionStatus(status: SessionStatus): boolean {
  return status.type === "running" || status.type === "input" || status.type === "idle"
}

export function isTerminalSessionStatus(status: SessionStatus): boolean {
  return status.type === "success" || status.type === "error" || status.type === "aborted"
}
```

### Background Task State (`src/features/background-agent/types.ts`)
```typescript
export type BackgroundTaskStatus =
  | { status: "pending"; queuedAt: number }
  | { status: "running"; sessionID: string; startedAt: number; parentSessionID?: string }
  | { status: "completed"; sessionID: string; completedAt: number; summary?: string }
  | { status: "error"; sessionID: string; error: string; errorAt: number }
  | { status: "cancelled"; sessionID: string; cancelledAt: number }
```

### Retry Status Processing (`src/plugin/event.ts`)
```typescript
if (sessionID && status?.type === "idle") {
  lastHandledRetryStatusKey.delete(sessionID)
}

if (sessionID && status?.type === "retry" && isModelFallbackEnabled && !isRuntimeFallbackEnabled) {
  // Handle retry state with deduplication
  const retryKey = `${retryAttempt}:${parsedForKey.providerID}/${parsedForKey.modelID}:${normalizeRetryStatusMessage(retryMessage)}`
  if (lastHandledRetryStatusKey.get(sessionID) === retryKey) {
    return  // Already handled this retry countdown
  }
  lastHandledRetryStatusKey.set(sessionID, retryKey)
  // ... trigger fallback
}
```

---

## 6. Template Method Pattern

**Purpose:** Define the skeleton of an algorithm, deferring some steps to subclasses.

**Evidence:**

### Hook Composition (`src/create-hooks.ts`)
```typescript
export function createHooks(args: {...}) {
  // Same structure for all hook types:
  const core = createCoreHooks({ /* ... */ })      // Step 1
  const continuation = createContinuationHooks({ /* ... */ })  // Step 2
  const skill = createSkillHooks({ /* ... */ })    // Step 3

  return { ...core, ...continuation, ...skill }
}
```

### Core Hooks Factory (`src/plugin/hooks/create-core-hooks.ts`)
```typescript
export function createCoreHooks(args: {
  ctx: PluginContext
  pluginConfig: OhMyOpenCodeConfig
  modelCacheState: ModelCacheState
  isHookEnabled: (hookName: HookName) => boolean
  safeHookEnabled: boolean
}) {
  // Same structure:
  const session = createSessionHooks({ /* ... */ })      // Sub-step 1
  const tool = createToolGuardHooks({ /* ... */ })       // Sub-step 2
  const transform = createTransformHooks({ /* ... */ })   // Sub-step 3

  return { ...session, ...tool, ...transform }
}
```

---

## 7. Proxy/Middleware Pattern

**Purpose:** Provide a surrogate or placeholder for another object to control access to it.

**Evidence:**

### Tool Execute After Middleware (`src/plugin/tool-execute-after.ts`)
```typescript
const runToolExecuteAfterHooks = async (): Promise<void> => {
  await hooks.toolOutputTruncator?.["tool.execute.after"]?.(input, output)
  await hooks.claudeCodeHooks?.["tool.execute.after"]?.(input, output)
  await hooks.preemptiveCompaction?.["tool.execute.after"]?.(input, output)
  await hooks.contextWindowMonitor?.["tool.execute.after"]?.(input, output)
  await hooks.commentChecker?.["tool.execute.after"]?.(input, output)
  await hooks.directoryAgentsInjector?.["tool.execute.after"]?.(input, output)
  // ... 15+ more hooks in chain
}

if (input.tool === "extract" || input.tool === "discard") {
  const originalOutput = { title: output.title, output: output.output, metadata: { ...output.metadata } }
  try {
    await runToolExecuteAfterHooks()
  } catch (error) {
    output.title = originalOutput.title
    output.output = originalOutput.output
    output.metadata = originalOutput.metadata
  }
  return
}
await runToolExecuteAfterHooks()
```

### Message Transform Hook (`src/plugin/messages-transform.ts`)
```typescript
export function createMessagesTransformHandler(args: { hooks: CreatedHooks }) {
  return async (input: unknown, output: unknown): Promise<void> => {
    // Transform passes through multiple hooks in order
    await hooks.sessionNotification?.(input as EventInput)
    await hooks.directoryAgentsInjector?.event?.(input as EventInput)
    await hooks.directoryReadmeInjector?.event?.(input as EventInput)
    // ... modifies messages as they pass through
  }
}
```

---

## 8. Registry Pattern

**Purpose:** Maintain a central registry of items that can be accessed from anywhere in the system.

**Evidence:**

### Session Category Registry (`src/shared/session-category-registry.ts`)
```typescript
const sessionCategoryMap = new Map<string, string>()

export const SessionCategoryRegistry = {
  register: (sessionID: string, category: string): void => {
    sessionCategoryMap.set(sessionID, category)
  },
  get: (sessionID: string): string | undefined => {
    return sessionCategoryMap.get(sessionID)
  },
  remove: (sessionID: string): void => {
    sessionCategoryMap.delete(sessionID)
  },
  has: (sessionID: string): boolean => {
    return sessionCategoryMap.has(sessionID)
  },
  size: (): number => sessionCategoryMap.size,
  clear: (): void => sessionCategoryMap.clear(),
}
```

### Tool Metadata Store (`src/features/tool-metadata-store.ts`)
```typescript
const toolMetadata = new Map<string, ToolMetadata>()

export const ToolMetadataStore = {
  set: (sessionID: string, callID: string, metadata: ToolMetadata) => {
    toolMetadata.set(`${sessionID}:${callID}`, metadata)
  },
  get: (sessionID: string, callID: string): ToolMetadata | undefined => {
    return toolMetadata.get(`${sessionID}:${callID}`)
  },
  consume: (sessionID: string, callID: string): ToolMetadata | undefined => {
    const key = `${sessionID}:${callID}`
    const value = toolMetadata.get(key)
    toolMetadata.delete(key)
    return value
  },
}
```

### Connected Providers Cache (`src/shared/connected-providers-cache.ts`)
```typescript
let cached: string[] | null = null

export function readConnectedProvidersCache(): string[] | null {
  return cached
}

export function writeConnectedProvidersCache(providers: string[]): void {
  cached = providers
}
```

---

## 9. Builder/Fluent Pattern

**Purpose:** Construct complex objects step by step, allowing the same construction process to create different representations.

**Evidence:**

### Agent Builder (`src/agents/agent-builder.ts`)
```typescript
export function buildAgent(
  source: AgentSource,
  model: string,
  categories?: CategoriesConfig,
  gitMasterConfig?: GitMasterConfig,
  browserProvider?: BrowserAutomationProvider,
  disabledSkills?: Set<string>
): AgentConfig {
  const base = isFactory(source) ? source(model) : { ...source }
  const categoryConfigs: Record<string, CategoryConfig> = mergeCategories(categories)

  const agentWithCategory = base as AgentConfig & { category?: string; skills?: string[]; variant?: string }
  if (agentWithCategory.category) {
    const categoryConfig = categoryConfigs[agentWithCategory.category]
    if (categoryConfig) {
      if (!base.model) base.model = categoryConfig.model
      if (base.temperature === undefined && categoryConfig.temperature !== undefined)
        base.temperature = categoryConfig.temperature
      if (base.variant === undefined && categoryConfig.variant !== undefined)
        base.variant = categoryConfig.variant
    }
  }

  if (agentWithCategory.skills?.length) {
    const { resolved } = resolveMultipleSkills(agentWithCategory.skills, { gitMasterConfig, browserProvider, disabledSkills })
    if (resolved.size > 0) {
      const skillContent = Array.from(resolved.values()).join("\n\n")
      base.prompt = skillContent + (base.prompt ? "\n\n" + base.prompt : "")
    }
  }

  return base
}
```

### Builtin Agents Factory (`src/agents/builtin-agents.ts`)
```typescript
export async function createBuiltinAgents(
  disabledAgents: string[] = [],
  agentOverrides: AgentOverrides = {},
  directory?: string,
  systemDefaultModel?: string,
  categories?: CategoriesConfig,
  gitMasterConfig?: GitMasterConfig,
  discoveredSkills: LoadedSkill[] = [],
  // ... many more optional parameters
): Promise<Record<string, AgentConfig>> {
  // Building happens in stages:
  const mergedCategories = mergeCategories(categories)
  const availableSkills = buildAvailableSkills(discoveredSkills, browserProvider, disabledSkills)
  const { pendingAgentConfigs, availableAgents } = collectPendingBuiltinAgents({ agentSources, agentMetadata, ... })
  // ... then sisyphus, hephaestus, atlas configs
  return result
}
```

---

## 10. Command Pattern

**Purpose:** Encapsulate a request as an object, thereby allowing parameterization of clients with different requests.

**Evidence:**

### CLI Entry (`src/cli/index.ts`)
```typescript
#!/usr/bin/env bun
import { runCli } from "./cli-program"
runCli()
```

### CLI Program Structure (`src/cli/cli-program.ts`)
```typescript
export async function runCli() {
  const program = new Command()
  program
    .name("oh-my-opencode")
    .description("AI Agent Harness for OpenCode")
    .addCommand(createInstallCommand())
    .addCommand(createDoctorCommand())
    .addCommand(createRunCommand())
    .addCommand(createConfigCommand())
    // ...
}
```

### Individual CLI Commands (`src/cli/`)
```
src/cli/
  install.ts      # Installation logic
  doctor/         # Diagnostic checks
  run/            # Run command
  config-manager/ # Config management
  skill/          # Skill management
  skill-mcp/      # Skill MCP integration
```

---

## 11. Dependency Injection

**Purpose:** Supply dependencies to an object rather than having it create them.

**Evidence:**

### Plugin Composition Root (`src/index.ts`)
```typescript
const OhMyOpenCodePlugin: Plugin = async (ctx) => {
  // Dependencies injected into factories
  const pluginConfig = loadPluginConfig(ctx.directory, ctx)

  const managers = createManagers({
    ctx,                      // Injected
    pluginConfig,              // Injected
    tmuxConfig,                // Derived
    modelCacheState,           // Created locally
    backgroundNotificationHookEnabled: isHookEnabled("background-notification"),
  })

  const toolsResult = await createTools({
    ctx,                       // Injected
    pluginConfig,              // Injected
    managers,                  // Injected from previous factory
  })

  const hooks = createHooks({
    ctx,                       // Injected
    pluginConfig,              // Injected
    modelCacheState,           // Shared state
    backgroundManager: managers.backgroundManager,  // From manager factory
    isHookEnabled,             // Derived from config
    safeHookEnabled,           // Derived from config
    mergedSkills: toolsResult.mergedSkills,
    availableSkills: toolsResult.availableSkills,
  })

  const pluginInterface = createPluginInterface({
    ctx,
    pluginConfig,
    firstMessageVariantGate,
    managers,                  // All managers
    hooks,                     // All hooks
    tools: toolsResult.filteredTools,
  })
}
```

---

## 12. Composition Pattern

**Purpose:** Compose objects into tree structures to represent part-whole hierarchies.

**Evidence:**

### Plugin Interface Composition (`src/plugin-interface.ts`)
```typescript
export function createPluginInterface(args: {...}): PluginInterface {
  const { ctx, pluginConfig, firstMessageVariantGate, managers, hooks, tools } = args

  return {
    tool: tools,

    "chat.params": async (input, output) => {
      const handler = createChatParamsHandler({ anthropicEffort: hooks.anthropicEffort, client: ctx.client })
      await handler(input, output)
    },

    "chat.headers": createChatHeadersHandler({ ctx }),

    "chat.message": createChatMessageHandler({ ctx, pluginConfig, firstMessageVariantGate, hooks }),

    "experimental.chat.messages.transform": createMessagesTransformHandler({ hooks }),

    "experimental.chat.system.transform": createSystemTransformHandler(),

    config: managers.configHandler,

    event: createEventHandler({ ctx, pluginConfig, firstMessageVariantGate, managers, hooks }),

    "tool.execute.before": createToolExecuteBeforeHandler({ ctx, hooks }),

    "tool.execute.after": createToolExecuteAfterHandler({ ctx, hooks }),
  }
}
```

### Hook Category Composition (`src/plugin/hooks/create-core-hooks.ts`)
```typescript
export function createCoreHooks(args: {...}) {
  const session = createSessionHooks({ ctx, pluginConfig, modelCacheState, isHookEnabled, safeHookEnabled })
  const tool = createToolGuardHooks({ ctx, pluginConfig, modelCacheState, isHookEnabled, safeHookEnabled })
  const transform = createTransformHooks({ ctx, pluginConfig, isHookEnabled, safeHookEnabled })

  return {
    ...session,   // Compose session hooks
    ...tool,      // Compose tool hooks
    ...transform, // Compose transform hooks
  }
}
```

---

## Pattern Usage Summary

| Pattern | Where Used | Purpose |
|---------|------------|---------|
| Factory | `createHooks`, `createManagers`, `createTools`, agent factories | Create instances without specifying concrete classes |
| Singleton | `SessionCategoryRegistry`, `ToolMetadataStore`, managers | Single instance access across module |
| Observer | `event.ts`, hook system | Decouple event producers from consumers |
| Strategy | `fallback-chain-from-models.ts`, error classifiers | Select algorithms at runtime |
| State Machine | `BackgroundManager`, session status | Manage complex state transitions |
| Template Method | `createCoreHooks`, `createContinuationHooks` | Define hook creation skeleton |
| Proxy/Middleware | `tool-execute-after`, transform hooks | Intercept and modify behavior |
| Registry | `SessionCategoryRegistry`, caches | Global access to shared data |
| Builder | `buildAgent`, `createBuiltinAgents` | Construct complex objects step by step |
| Command | CLI commands | Encapsulate operations as objects |
| Dependency Injection | `index.ts` composition root | Supply dependencies externally |
| Composition | `createPluginInterface`, hook factories | Build complex interfaces from parts |
