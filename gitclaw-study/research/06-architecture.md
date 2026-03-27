# GitClaw Architecture Analysis

## Architectural Pattern

**Primary Pattern: Hexagonal (Ports & Adapters) with Layered Concerns**

GitClaw follows a hexagonal architecture where the core agent is insulated from infrastructure concerns. The `Agent` from `@mariozechner/pi-agent-core` is the domain center, surrounded by ports (tool interface, hook interface, plugin interface) and adapters (CLI, SDK, voice, sandbox).

```
┌─────────────────────────────────────────────────────────────┐
│                     CLI / SDK / Voice                        │
│                  (Primary Adapters)                          │
├─────────────────────────────────────────────────────────────┤
│                    Agent Core                                │
│  ┌─────────┐  ┌─────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ Loader  │  │  Hooks  │  │ Plugins  │  │   Skills     │  │
│  └─────────┘  └─────────┘  └──────────┘  └──────────────┘  │
│  ┌─────────┐  ┌─────────┐  ┌──────────┐  ┌──────────────┐  │
│  │Workflows│  │ Tools   │  │ Sandbox  │  │   Learning   │  │
│  └─────────┘  └─────────┘  └──────────┘  └──────────────┘  │
├─────────────────────────────────────────────────────────────┤
│              pi-agent-core (Domain Center)                   │
│           (Agent, Model, Tool Execution)                     │
└─────────────────────────────────────────────────────────────┘
```

**Secondary Patterns Observed:**

| Pattern | Location | Usage |
|---------|----------|-------|
| Factory | `createBuiltinTools()`, `createSandboxContext()`, `createPluginApi()` | Object creation with conditional logic |
| Decorator | `wrapToolWithHooks()`, `wrapToolWithProgrammaticHooks()` | Adds behavior to tools without modifying them |
| Observer | `agent.subscribe(handler)` | Decoupled event notification |
| Strategy | `OpenAIRealtimeAdapter`, `GeminiLiveAdapter` | Interchangeable voice backends |
| Pipeline | Hook chain: `pre_tool_use` -> block/modify/allow | Processing chain with early exit |
| Channel | `createChannel<T>()` in SDK | AsyncIterator-based message streaming |
| Repository | `discoverSkills()`, `discoverWorkflows()`, `discoverAndLoadPlugins()` | File-based discovery |

---

## Component Map

### Entry Points

```
CLI Entry (index.ts)
    ├── REPL loop (readline)
    ├── Voice server mode (--voice flag)
    └── Plugin subcommand (gitclaw plugin ...)

SDK Entry (exports.ts / sdk.ts)
    ├── query() -> AsyncGenerator<GCMessage>
    └── tool() -> GCToolDefinition (helper)
```

### Core Modules

| Module | File | Responsibility |
|--------|------|----------------|
| **Loader** | `src/loader.ts` | Parses agent.yaml, resolves inheritance, builds system prompt |
| **Tools** | `src/tools/index.ts` | Factory for built-in tools (cli, read, write, memory, etc.) |
| **Tool Loader** | `src/tool-loader.ts` | Declarative tool loading from YAML |
| **Skills** | `src/skills.ts` | Skill discovery, loading, formatting for prompt |
| **Workflows** | `src/workflows.ts` | Workflow discovery and flow execution |
| **Hooks** | `src/hooks.ts` | Lifecycle hook execution (on_session_start, pre_tool_use, post_response, on_error) |
| **Plugins** | `src/plugins.ts` | Plugin discovery, loading, merging |
| **Plugin SDK** | `src/plugin-sdk.ts` | API exposed to plugin register() functions |
| **Sandbox** | `src/sandbox.ts` | E2B sandbox context via gitmachine |
| **Learning** | `src/learning/reinforcement.ts` | Confidence tracking, skill crystallization |
| **Audit** | `src/audit.ts` | JSONL audit logging |
| **Compliance** | `src/compliance.ts` | Risk validation, regulatory framework checks |
| **Context** | `src/context.ts` | Memory/summary/chat aggregation for voice LLM |
| **Voice** | `src/voice/server.ts` | WebSocket voice server |
| **Voice Adapters** | `src/voice/openai-realtime.ts`, `gemini-live.ts` | Provider-specific WebRTC/audio handling |

---

## Key Abstractions and Interfaces

### AgentTool Interface
```typescript
interface AgentTool<T = any> {
  name: string;
  label: string;
  description: string;
  parameters: T; // Typebox schema
  execute(
    toolCallId: string,
    args: any,
    signal?: AbortSignal,
    onUpdate?: AgentToolUpdateCallback,
  ): Promise<{ content: [{type: "text", text: string}]; details?: any }>;
}
```

### Hook System
```typescript
interface HookDefinition {
  script: string;
  description?: string;
  baseDir?: string;
  _handler?: (ctx: Record<string, any>) => Promise<HookResult> | HookResult;
}

interface HookResult {
  action: "allow" | "block" | "modify";
  reason?: string;
  args?: Record<string, any>;  // for modify action
}
```

### Plugin Contract (plugin.yaml schema)
```typescript
interface PluginManifest {
  id: string;           // kebab-case
  name: string;
  version: string;
  description: string;
  provides?: {
    tools?: boolean;
    hooks?: Record<HookEvent, Array<{script: string; description?: string}>>;
    skills?: boolean;
    prompt?: string;
  };
  config?: { properties?: Record<string, PluginConfigProperty>; required?: string[] };
  entry?: string;       // programmatic entry point
  engine?: string;      // min gitclaw version
}
```

### GitclawPluginApi (for programmatic plugins)
```typescript
interface GitclawPluginApi {
  pluginId: string;
  pluginDir: string;
  config: Record<string, any>;
  registerTool(def: GCToolDefinition): void;
  registerHook(event: HookEvent, handler: HookHandler): void;
  addPrompt(text: string): void;
  registerMemoryLayer(layer: {name: string; path: string; description: string}): void;
  logger: { info/warn/error };
}
```

### SDK Query Interface
```typescript
interface Query extends AsyncGenerator<GCMessage, void, undefined> {
  abort(): void;
  steer(message: string): void;
  sessionId(): string;
  manifest(): AgentManifest;
  messages(): GCMessage[];
}
```

---

## Component Communication

### Agent Orchestration Flow

```
User Input (CLI/SDK/Voice)
        │
        ▼
┌─────────────────────────────────┐
│         Agent (pi-agent-core)  │
│  ┌───────────────────────────┐  │
│  │ System Prompt + Tools    │  │
│  └───────────────────────────┘  │
│            │
│            ▼ (tool calls)
│  ┌─────────────────────────────────┐
│  │ Tool Execution (hooks wrapped)  │
│  │  1. pre_tool_use hooks          │
│  │  2. Tool.execute()             │
│  │  3. post_response hooks         │
│  └─────────────────────────────────┘
│            │
│            ▼ (events)
│  ┌─────────────────────────────────┐
│  │ Event Subscribers               │
│  │  - CLI: handleEvent()          │
│  │  - SDK: channel.push()         │
│  │  - Voice: adapter.onMessage()   │
│  └─────────────────────────────────┘
```

### System Prompt Assembly (loader.ts)

The loader composes the system prompt from multiple sources in priority order:

1. `manifest.name` / `manifest.description`
2. `SOUL.md` (identity)
3. `RULES.md` (behavioral constraints)
4. Parent rules (via `extends` inheritance)
5. `DUTIES.md` (responsibilities)
6. `AGENTS.md` (sub-agents)
7. Memory instruction block
8. Knowledge block
9. Skills block
10. Workflows block
11. Sub-agents block
12. Examples block
13. Plugin prompt additions
14. Compliance context
15. Workspace directory instruction
16. Task learning instruction

### Plugin Loading Pipeline

```
agent.yaml (plugins config)
        │
        ▼
discoverAndLoadPlugins()
        │
        ├── Auto-install from source (git clone)
        │
        ▼
discoverPluginDirs()  [local → global → installed]
        │
        ▼
loadPlugin()
        │
        ├── validatePluginManifest()
        ├── check engine compatibility
        ├── resolvePluginConfig()  [user config → env → default]
        ├── loadDeclarativeTools()
        ├── loadHooks()
        ├── discoverSkills()
        ├── loadPrompt()
        └── import programmatic entry (register())
        │
        ▼
mergeHooksConfigs()  [agent hooks + plugin hooks]
        │
        ▼
Tools + Hooks ready for Agent
```

---

## Design Patterns in Detail

### 1. Decorator Pattern (Hooks on Tools)

**File:** `src/hooks.ts`

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
      return originalExecute(toolCallId, finalArgs, signal, onUpdate);
    },
  } as T;
}
```

### 2. Factory Pattern (Tool Creation)

**File:** `src/tools/index.ts`

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

### 3. Strategy Pattern (Voice Adapters)

**Files:** `src/voice/openai-realtime.ts`, `src/voice/gemini-live.ts`

```typescript
interface MultimodalAdapter {
  connect(opts: {
    toolHandler: (query: string) => Promise<string>;
    onMessage: (msg: ServerMessage) => void;
  }): Promise<void>;
  send(msg: ClientMessage): void;
  disconnect(): Promise<void>;
}
```

Both OpenAI and Gemini implement the same interface, allowing runtime switching.

### 4. Observer Pattern (Agent Events)

**File:** `src/index.ts`

```typescript
agent.subscribe((event: AgentEvent) => handleEvent(event, ...));

function handleEvent(event: AgentEvent) {
  switch (event.type) {
    case "message_update":    // streaming text
    case "message_end":       // complete response
    case "tool_execution_start":
    case "tool_execution_end":
    case "agent_end":
  }
}
```

### 5. Channel Pattern (SDK Streaming)

**File:** `src/sdk.ts`

```typescript
function createChannel<T>(): Channel<T> {
  const buffer: T[] = [];
  let resolve: ((v: IteratorResult<T>) => void) | null = null;
  let done = false;

  return {
    push(v: T) { /* queue value */ },
    finish() { done = true; /* release waiting iterator */ },
    pull(): Promise<IteratorResult<T>> { /* async dequeue */ },
  };
}
```

The SDK's `query()` returns an AsyncGenerator that consumers iterate over to receive streaming messages.

### 6. Repository Pattern (File Discovery)

**Pattern used across:** skills, workflows, plugins, tools

```typescript
async function discoverSkills(agentDir: string): Promise<SkillMetadata[]> {
  const skillsDir = join(agentDir, "skills");
  // ... verify directory
  for (const entry of entries) {
    if (!entry.isDirectory()) continue;
    const content = await readFile(join(skillDir, "SKILL.md"), "utf-8");
    const { frontmatter } = parseFrontmatter(content);
    // validate and return metadata
  }
}
```

---

## Memory System

### Git-Backed Memory Pattern

**File:** `src/tools/memory.ts`

Memory is stored in `memory/MEMORY.md` and committed to git on every save:

```typescript
// Save memory
await writeFile(memoryFile, content, "utf-8");
execSync(`git add "${memoryPath}" && git commit -m "${commitMsg}"`);
```

This gives:
- Full history via `git log`
- Diffs via `git diff`
- Attribution via commit authorship
- Archive via `git push`

### Layered Memory

The memory tool supports multiple layers defined in `memory/memory.yaml`:

```yaml
layers:
  - name: working
    path: memory/MEMORY.md
    max_lines: 500
  - name: archive
    path: memory/archive/{year-month}.md
```

---

## Learning System

### Confidence-Based Skill Learning

**File:** `src/learning/reinforcement.ts`

Skills track confidence based on usage outcomes:

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

Negative examples are stored to avoid repeating failed approaches.

---

## Execution Modes

### CLI Mode (src/index.ts)
- REPL with readline interface
- Slash commands: `/skills`, `/plugins`, `/memory`, `/tasks`, `/learned`, `/quit`
- Skill expansion: `/skill:name [args]`
- Single-shot mode: `--prompt "..."`

### SDK Mode (src/sdk.ts)
- `query(options)` returns AsyncGenerator
- Full control over tools, hooks, model constraints
- Multi-turn via async iterable input

### Voice Mode (src/voice/server.ts)
- WebSocket server on configurable port
- Browser UI (ui.html) connects via WebSocket
- Two adapter backends: OpenAI Realtime, Gemini Live
- Context injection: memory + summary + recent chat

### Sandbox Mode
- E2B sandbox via gitmachine
- Full VM isolation
- Git operations via gitmachine API
- Used for dangerous operations

---

## Configuration Schema

### agent.yaml

```yaml
spec_version: "0.1.0"
name: string
version: string
description: string
model:
  preferred: "provider:model-id"  # e.g., "anthropic:claude-sonnet-4-5-20250929"
  fallback: ["provider:model-id"]
  constraints?: { temperature?, max_tokens?, top_p?, top_k?, stop_sequences? }
tools: string[]  # tool names
skills?: string[]  # allowed skill names (empty = all)
runtime:
  max_turns: number
  timeout?: number
extends?: string  # git URL for parent agent
dependencies?: Array<{ name, source, version, mount }>
delegation?: { mode: "auto" | "explicit" | "router", router?: string }
compliance?: ComplianceConfig
plugins?: Record<pluginName, PluginConfig>
```

---

## Extension Points

### Adding a New Built-in Tool
1. Create `src/tools/mytool.ts` with `createMyTool()` factory
2. Export from `src/tools/index.ts`
3. Add to `createBuiltinTools()` return array

### Adding a New Voice Adapter
1. Implement `MultimodalAdapter` interface
2. Add adapter to `src/voice/server.ts` import
3. Add case in voice mode initialization

### Adding Plugin Capabilities
- Tools: Define in `plugin.yaml` provides.tools
- Hooks: Define in `plugin.yaml` provides.hooks
- Skills: Create `skills/` directory with SKILL.md files
- Prompt: Set `provides.prompt` path
- Programmatic: Export `register(api)` function

---

## Summary

GitClaw is a **composition-focused** architecture rather than a framework. The agent core (`pi-agent-core`) handles LLM interaction and tool orchestration, while GitClaw layers on:

1. **Configuration-driven setup** (YAML-based agent specs)
2. **Extensible plugin system** (hooks, tools, skills, prompts)
3. **Multiple entry points** (CLI, SDK, Voice)
4. **Git-native storage** (memory, audit, versioning)
5. **Learning-based improvement** (confidence tracking)

The hexagonal ports (AgentTool, HookDefinition, PluginManifest) ensure the core remains portable while adapters (CLI, SDK, Voice, Sandbox) handle environment-specific concerns.
