# Design Patterns - GitClaw

## Pattern Inventory

| # | Pattern | Location | Evidence |
|---|---------|----------|----------|
| 1 | Decorator | `src/hooks.ts` | wrapToolWithHooks() wraps tool.execute with hook chain |
| 2 | Factory | `src/tools/index.ts`, `src/sandbox.ts`, `src/plugin-sdk.ts` | createBuiltinTools(), createSandboxContext(), createPluginApi() |
| 3 | Observer | `src/index.ts` | agent.subscribe(handler) for event notification |
| 4 | Strategy | `src/voice/openai-realtime.ts`, `src/voice/gemini-live.ts` | Interchangeable MultimodalAdapter implementations |
| 5 | Pipeline | `src/hooks.ts` | Hook chain: pre_tool_use -> block/modify/allow with early exit |
| 6 | Channel | `src/sdk.ts` | createChannel<T>() for AsyncIterator-based streaming |
| 7 | Repository | `src/skills.ts`, `src/workflows.ts`, `src/plugins.ts` | File-based discovery with metadata parsing |
| 8 | Singleton | `src/config.ts` | Global config instance |
| 9 | Wrapper | `src/sandbox.ts` | SandboxContext wraps gitmachine for safe git operations |

---

## 1. Decorator Pattern

**File:** `src/hooks.ts`

**Purpose:** Adds behavior to tools without modifying their core implementation. Hooks intercept tool execution for pre-processing and post-processing.

**Evidence:**

```typescript
function wrapToolWithHooks<T extends AgentTool<any>>(
  tool: T,
  hooksConfig: HooksConfig,
  agentDir: string,
  sessionId: string,
): T {
  const originalExecute = tool.execute;
  return {
    ...tool,
    execute: async (toolCallId, args, signal, onUpdate) => {
      // pre_tool_use hook
      const result = await runHooks(preToolHooks, agentDir, {...});
      if (result.action === "block") throw new Error("blocked");
      const finalArgs = result.action === "modify" ? result.args : args;

      // Call original with potentially modified args
      const execResult = await originalExecute(toolCallId, finalArgs, signal, onUpdate);

      // post_response hooks
      await runHooks(postResponseHooks, agentDir, {...});

      return execResult;
    },
  } as T;
}
```

**Usage:** All tools are wrapped before being passed to the agent, enabling hooks for auditing, validation, and modification.

---

## 2. Factory Pattern

**File:** `src/tools/index.ts`

**Purpose:** Centralized object creation with conditional logic based on configuration.

**Evidence:**

```typescript
export function createBuiltinTools(config: BuiltinToolsConfig): AgentTool<any>[] {
  if (config.sandbox) {
    return [
      createSandboxCliTool(config.sandbox, config.timeout),
      createSandboxReadTool(config.sandbox),
      createSandboxWriteTool(config.sandbox),
      createSandboxMemoryTool(config.sandbox),
    ];
  }
  return [
    createCliTool(config.dir, config.timeout),
    createReadTool(config.dir),
    createWriteTool(config.dir),
    createMemoryTool(config.dir, config.pluginMemoryLayers),
    createCapturePhotoTool(config.dir),
  ];
}
```

**Also in:**
- `src/sandbox.ts`: `createSandboxContext()` returns configured SandboxContext
- `src/plugin-sdk.ts`: `createPluginApi()` returns GitclawPluginApi instance

---

## 3. Observer Pattern

**File:** `src/index.ts`

**Purpose:** Decoupled event notification. The agent emits events and multiple subscribers can react independently.

**Evidence:**

```typescript
agent.subscribe((event: AgentEvent) => handleEvent(event, ...));

function handleEvent(event: AgentEvent) {
  switch (event.type) {
    case "message_update":    // streaming text
    case "message_end":       // complete response
    case "tool_execution_start":
    case "tool_execution_end":
    case "agent_end":
    case "error":
      // React to event
  }
}
```

**Usage:** CLI, SDK, and Voice adapters all subscribe to the same agent events but handle them differently.

---

## 4. Strategy Pattern

**Files:** `src/voice/openai-realtime.ts`, `src/voice/gemini-live.ts`

**Purpose:** Interchangeable voice backends. The same interface supports different AI provider implementations.

**Evidence:**

```typescript
interface MultimodalAdapter {
  connect(opts: {
    toolHandler: (query: string) => Promise<string>;
    onMessage: (msg: ServerMessage) => void;
  }): Promise<void>;
  send(msg: ClientMessage): void;
  disconnect(): Promise<void>;
}

// OpenAI implementation
export class OpenAIRealtimeAdapter implements MultimodalAdapter { ... }

// Gemini implementation
export class GeminiLiveAdapter implements MultimodalAdapter { ... }
```

**Usage:** Voice mode in `src/voice/server.ts` selects adapter based on configuration without changing call sites.

---

## 5. Pipeline Pattern

**File:** `src/hooks.ts`

**Purpose:** Sequential processing with early exit. Hooks form a chain where each can block, modify, or allow continuation.

**Evidence:**

```typescript
interface HookResult {
  action: "allow" | "block" | "modify";
  reason?: string;
  args?: Record<string, any>;  // modified args for modify action
}

async function runHooks(hooks: HookDefinition[], agentDir: string, ctx: Record<string, any>): Promise<HookResult> {
  for (const hook of hooks) {
    const result = await executeHook(hook, ctx);
    if (result.action !== "allow") {
      return result;  // Early exit on block or modify
    }
  }
  return { action: "allow" };
}
```

**Hook Events:**
- `on_session_start`
- `pre_tool_use`
- `post_response`
- `on_error`

---

## 6. Channel Pattern

**File:** `src/sdk.ts`

**Purpose:** AsyncIterator-based message streaming. Enables cooperative multitasking for SDK consumers.

**Evidence:**

```typescript
function createChannel<T>(): Channel<T> {
  const buffer: T[] = [];
  let resolve: ((v: IteratorResult<T>) => void) | null = null;
  let done = false;

  return {
    push(v: T) {
      if (resolve) {
        resolve({ value: v, done: false });
        resolve = null;
      } else {
        buffer.push(v);
      }
    },
    finish() {
      done = true;
      if (resolve) {
        resolve({ value: undefined, done: true });
      }
    },
    pull(): Promise<IteratorResult<T>> {
      if (buffer.length > 0) {
        return Promise.resolve({ value: buffer.shift()!, done: false });
      }
      if (done) {
        return Promise.resolve({ value: undefined, done: true });
      }
      return new Promise(r => { resolve = r; });
    },
    [Symbol.asyncIterator]() { return this; },
    next() { return this.pull(); }
  };
}
```

**Usage:** SDK's `query()` returns an AsyncGenerator. Consumers iterate to receive streaming messages.

---

## 7. Repository Pattern

**Files:** `src/skills.ts`, `src/workflows.ts`, `src/plugins.ts`

**Purpose:** File-based discovery with metadata extraction. Centralizes how entities are located and hydrated.

**Evidence:**

```typescript
async function discoverSkills(agentDir: string): Promise<SkillMetadata[]> {
  const skillsDir = join(agentDir, "skills");
  if (!existsSync(skillsDir)) return [];

  const entries = await readdir(skillsDir, { withFileTypes: true });
  const skills: SkillMetadata[] = [];

  for (const entry of entries) {
    if (!entry.isDirectory()) continue;
    const content = await readFile(join(skillsDir, entry.name, "SKILL.md"), "utf-8");
    const { frontmatter } = parseFrontmatter(content);
    // Validate and return metadata
  }
  return skills;
}
```

**Also in:**
- `discoverWorkflows()` - reads workflow YAML files
- `discoverAndLoadPlugins()` - finds plugin directories, validates manifest

---

## 8. Singleton Pattern

**File:** `src/config.ts`

**Purpose:** Global configuration state accessed throughout the application.

**Evidence:**

```typescript
let config: GitClawConfig | undefined;

export function getConfig(): GitClawConfig {
  if (!config) {
    config = loadConfig();  // Lazy initialization
  }
  return config;
}
```

---

## 9. Wrapper Pattern

**File:** `src/sandbox.ts`

**Purpose:** Wraps gitmachine API with simplified, safer git operations for agent use.

**Evidence:**

```typescript
export class SandboxContext {
  constructor(
    private gitmachine: GitMachineClient,
    private timeout: number = 60000,
  ) {}

  async execGit(args: string[]): Promise<GitResult> {
    // Simplified git operation interface
  }

  async readFile(path: string): Promise<string> {
    // Safe file read with bounds checking
  }

  async writeFile(path: string, content: string): Promise<void> {
    // Safe file write
  }
}
```

---

## Git-Native Patterns

### Git-Backed Memory (src/tools/memory.ts)

Memory stored in `memory/MEMORY.md`, committed to git on save:

```typescript
await writeFile(memoryFile, content, "utf-8");
execSync(`git add "${memoryPath}" && git commit -m "${commitMsg}"`);
```

Benefits: Full history via `git log`, diffs via `git diff`, attribution via commit authorship.

### Layered Memory

```yaml
layers:
  - name: working
    path: memory/MEMORY.md
    max_lines: 500
  - name: archive
    path: memory/archive/{year-month}.md
```

---

## Learning Patterns

### Confidence-Based Skill Learning (src/learning/reinforcement.ts)

Asymptotic confidence growth with penalty for failures:

```typescript
function adjustConfidence(
  current: SkillStats,
  outcome: "success" | "failure" | "partial",
): SkillStats {
  switch (outcome) {
    case "success":
      // Asymptotic to 1.0: conf + 0.1 * (1 - conf)
      stats.confidence = Math.min(1.0, stats.confidence + 0.1 * (1 - stats.confidence));
    case "failure":
      // 2x penalty
      stats.confidence = Math.max(0.0, stats.confidence - 0.2);
    case "partial":
      stats.confidence = Math.max(0.0, stats.confidence - 0.05);
  }
}
```

---

## Pattern Relationship Map

```
Factory (createBuiltinTools)
    │
    ├── creates ──> AgentTool instances
    │
    └── wrapped by ──> Decorator (wrapToolWithHooks)
                           │
                           └── uses ──> Pipeline (hook chain)
                                            │
                                            └── built from ──> Repository (discoverHooks)

Strategy (MultimodalAdapter)
    │
    └── implemented by ──> OpenAIRealtimeAdapter, GeminiLiveAdapter

Channel (createChannel)
    │
    └── used by ──> SDK Query interface

Observer (agent.subscribe)
    │
    └── handles events from ──> Agent core
```
