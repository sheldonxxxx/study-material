# Architecture - GitClaw

## Architectural Pattern

**Primary: Hexagonal (Ports & Adapters) with Layered Concerns**

The agent core (`@mariozechner/pi-agent-core`) is the domain center, insulated from infrastructure. Ports define interfaces (tool, hook, plugin) while adapters handle environment-specific concerns.

```
CLI / SDK / Voice (Primary Adapters)
         │
         ▼
┌────────────────────────────────────────┐
│              Agent Core                │
│  ┌─────────┐ ┌─────────┐ ┌──────────┐ │
│  │ Loader  │ │  Hooks  │ │ Plugins   │ │
│  └─────────┘ └─────────┘ └──────────┘ │
│  ┌─────────┐ ┌─────────┐ ┌──────────┐ │
│  │Workflows│ │ Tools   │ │ Learning  │ │
│  └─────────┘ └─────────┘ └──────────┘ │
├────────────────────────────────────────┤
│       pi-agent-core (Domain Center)    │
└────────────────────────────────────────┘
```

---

## Module Responsibilities

### Core Modules

| Module | File | Responsibility |
|--------|------|----------------|
| **Loader** | `src/loader.ts` | Parses agent.yaml, resolves inheritance, builds system prompt from SOUL.md, RULES.md, skills, workflows |
| **Tools** | `src/tools/index.ts` | Factory for built-in tools (cli, read, write, memory) with sandbox variants |
| **Tool Loader** | `src/tool-loader.ts` | Declarative tool loading from YAML configuration |
| **Skills** | `src/skills.ts` | Skill discovery from `skills/` directory, loading SKILL.md metadata |
| **Workflows** | `src/workflows.ts` | Workflow discovery and sequential flow execution |
| **Hooks** | `src/hooks.ts` | Lifecycle hooks: on_session_start, pre_tool_use, post_response, on_error |
| **Plugins** | `src/plugins.ts` | Plugin discovery from local/global/installed dirs, loading declarative + programmatic plugins |
| **Plugin SDK** | `src/plugin-sdk.ts` | API exposed to plugin register() functions |
| **Sandbox** | `src/sandbox.ts` | E2B sandbox context via gitmachine for isolated execution |
| **Learning** | `src/learning/reinforcement.ts` | Confidence tracking, skill crystallization based on outcome feedback |
| **Audit** | `src/audit.ts` | JSONL audit logging for all agent operations |
| **Compliance** | `src/compliance.ts` | Risk validation and regulatory framework checks |
| **Context** | `src/context.ts` | Memory/summary/chat aggregation for voice LLM context injection |
| **Voice** | `src/voice/server.ts` | WebSocket voice server with OpenAI/Gemini realtime adapters |
| **Voice Adapters** | `src/voice/openai-realtime.ts`, `src/voice/gemini-live.ts` | Provider-specific WebRTC/audio handling implementing MultimodalAdapter interface |

### Entry Points

| Entry | File | Purpose |
|-------|------|---------|
| **CLI** | `src/index.ts` | Readline REPL interface, slash commands, voice server mode, plugin subcommands |
| **SDK** | `src/exports.ts`, `src/sdk.ts` | Library interface: query() returns AsyncGenerator<GCMessage>, tool() helper |

---

## Communication Patterns

### Agent Orchestration Flow

```
User Input (CLI/SDK/Voice)
         │
         ▼
┌─────────────────────────────────┐
│      Agent (pi-agent-core)      │
│  System Prompt + Tools           │
│         │
         ▼ (tool calls)
┌─────────────────────────────────┐
│  Tool Execution (hooks wrapped)  │
│  1. pre_tool_use hooks          │
│  2. Tool.execute()             │
│  3. post_response hooks         │
└─────────────────────────────────┘
         │
         ▼ (events)
┌─────────────────────────────────┐
│  Event Subscribers              │
│  - CLI: handleEvent()          │
│  - SDK: channel.push()         │
│  - Voice: adapter.onMessage()  │
└─────────────────────────────────┘
```

### System Prompt Assembly (loader.ts)

Priority-ordered composition:
1. manifest.name / manifest.description
2. SOUL.md (identity)
3. RULES.md (behavioral constraints)
4. Parent rules (via `extends` inheritance)
5. DUTIES.md (responsibilities)
6. AGENTS.md (sub-agents)
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
         ▼
discoverPluginDirs() [local → global → installed]
         │
         ▼
loadPlugin()
         │
         ├── validatePluginManifest()
         ├── check engine compatibility
         ├── resolvePluginConfig() [user config → env → default]
         ├── loadDeclarativeTools()
         ├── loadHooks()
         ├── discoverSkills()
         ├── loadPrompt()
         └── import programmatic entry (register())
         │
         ▼
mergeHooksConfigs() [agent hooks + plugin hooks]
         │
         ▼
Tools + Hooks ready for Agent
```

---

## Key Interfaces

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

### Plugin Contract

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

### GitclawPluginApi (programmatic plugins)

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

### Voice Adapter Interface

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

---

## Execution Modes

| Mode | Entry | Characteristics |
|------|-------|-----------------|
| **CLI** | `src/index.ts` | REPL with readline, slash commands (/skills, /plugins, /memory, /tasks), skill expansion /skill:name |
| **SDK** | `src/sdk.ts` | query() returns AsyncGenerator, full tool/hook/model control, multi-turn via async iterable |
| **Voice** | `src/voice/server.ts` | WebSocket server, browser UI (ui.html), OpenAI Realtime or Gemini Live backends |
| **Sandbox** | `src/sandbox.ts` | E2B sandbox via gitmachine, full VM isolation, git operations via gitmachine API |

---

## Configuration Schema

### agent.yaml

```yaml
spec_version: "0.1.0"
name: string
version: string
description: string
model:
  preferred: "provider:model-id"
  fallback: ["provider:model-id"]
  constraints?: { temperature?, max_tokens?, top_p?, top_k?, stop_sequences? }
tools: string[]
skills?: string[]  # allowed skill names (empty = all)
runtime:
  max_turns: number
  timeout?: number
extends?: string   # git URL for parent agent
dependencies?: Array<{ name, source, version, mount }>
delegation?: { mode: "auto" | "explicit" | "router", router?: string }
compliance?: ComplianceConfig
plugins?: Record<pluginName, PluginConfig>
```

---

## Extension Points

| Extension | Mechanism |
|-----------|------------|
| **New Built-in Tool** | Create `src/tools/mytool.ts` with createMyTool() factory, export from `src/tools/index.ts`, add to createBuiltinTools() |
| **New Voice Adapter** | Implement MultimodalAdapter interface, add to `src/voice/server.ts` import, add case in voice mode initialization |
| **Plugin Tools** | Define in plugin.yaml provides.tools |
| **Plugin Hooks** | Define in plugin.yaml provides.hooks |
| **Plugin Skills** | Create skills/ directory with SKILL.md files |
| **Plugin Prompt** | Set provides.prompt path in plugin.yaml |
| **Programmatic Plugin** | Export register(api) function from plugin entry |

---

## Architectural Summary

GitClaw is a **composition-focused** architecture rather than a framework:

1. **Configuration-driven setup** - YAML-based agent specs with inheritance
2. **Extensible plugin system** - hooks, tools, skills, prompts via plugin.yaml
3. **Multiple entry points** - CLI, SDK, Voice modes
4. **Git-native storage** - memory, audit, versioning via git commits
5. **Learning-based improvement** - confidence tracking with asymptotic growth

Hexagonal ports (AgentTool, HookDefinition, PluginManifest) ensure portability while adapters (CLI, SDK, Voice, Sandbox) handle environment-specific concerns.
