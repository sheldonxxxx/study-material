# Feature Deep Dive: Batch 1 (Core Features)

**Date:** 2026-03-26
**Repository:** `/Users/sheldon/Documents/claw/reference/opencode`
**Features Analyzed:**
1. AI Coding Agent (Core Engine)
2. Multi-Provider AI Support
3. Built-in Agent System (build/plan/general)

---

## Feature 1: AI Coding Agent (Core Engine)

### Overview

The AI Coding Agent is the central orchestrator that coordinates AI interactions, tool execution, and context management. It's built on the Effect framework for dependency injection and service composition.

### Core Implementation

**Primary File:** `packages/opencode/src/agent/agent.ts`

The agent system uses a **Service Layer pattern** with Effect:

```typescript
export namespace Agent {
  // Info schema defines agent configuration
  export const Info = z.object({
    name: z.string(),
    description: z.string().optional(),
    mode: z.enum(["subagent", "primary", "all"]),
    native: z.boolean().optional(),
    hidden: z.boolean().optional(),
    topP: z.number().optional(),
    temperature: z.number().optional(),
    color: z.string().optional(),
    permission: Permission.Ruleset,
    model: z.object({...}).optional(),
    variant: z.string().optional(),
    prompt: z.string().optional(),
    options: z.record(z.string(), z.any()),
    steps: z.number().int().positive().optional(),
  })

  export class Service extends ServiceMap.Service<Service, Interface>()("@opencode/Agent") {}

  export const layer = Layer.effect(
    Service,
    Effect.gen(function* () {
      // Agent state initialization with permissions, defaults, and custom agents
    })
  )
}
```

### Architecture Highlights

**1. Effect-Based Dependency Injection**

The agent uses `effect` framework's `ServiceMap` and `Layer` for composable dependency injection:

```typescript
import { Effect, ServiceMap, Layer } from "effect"
import { InstanceState } from "@/effect/instance-state"
import { makeRuntime } from "@/effect/run-service"

export class Service extends ServiceMap.Service<Service, Interface>()("@opencode/Agent") {}

// Runtime creation for sync access
const { runPromise } = makeRuntime(Service, defaultLayer)

export async function get(agent: string) {
  return runPromise((svc) => svc.get(agent))
}
```

**2. Permission System Integration**

Agents have fine-grained permission rules merged from defaults and user config:

```typescript
const defaults = Permission.fromConfig({
  "*": "allow",
  doom_loop: "ask",
  external_directory: {
    "*": "ask",
    ...Object.fromEntries(whitelistedDirs.map((dir) => [dir, "allow"])),
  },
  question: "deny",
  plan_enter: "deny",
  plan_exit: "deny",
  read: {
    "*": "allow",
    "*.env": "ask",
    "*.env.*": "ask",
    "*.env.example": "allow",
  },
})
```

**3. Agent Generation from Descriptions**

The agent can dynamically generate new agents from natural language descriptions using `generateObject` from the AI SDK:

```typescript
export async function generate(input: {
  description: string
  model?: { providerID: ProviderID; modelID: ModelID }
}) {
  const params = {
    temperature: 0.3,
    messages: [
      { role: "system", content: PROMPT_GENERATE },
      { role: "user", content: `Create an agent configuration based on: "${input.description}"...` }
    ],
    model: language,
    schema: z.object({
      identifier: z.string(),
      whenToUse: z.string(),
      systemPrompt: z.string(),
    }),
  }
  return yield* Effect.promise(() => generateObject(params).then((r) => r.object))
}
```

### Clever Solutions

1. **Plugin System Hook**: Agents can trigger experimental hooks for system prompt transformation:
   ```typescript
   yield* Effect.promise(() =>
     Plugin.trigger("experimental.chat.system.transform", { model: resolved }, { system }),
   )
   ```

2. **Smart Truncate.GLOB Allowlisting**: External directory access is automatically allowed for skill directories:
   ```typescript
   const whitelistedDirs = [Truncate.GLOB, ...skillDirs.map((dir) => path.join(dir, "*"))]
   ```

3. **Instance State Caching**: Uses `InstanceState.make` with `Effect.fn` for efficient memoized state access.

---

## Feature 2: Multi-Provider AI Support

### Overview

Provider-agnostic architecture supporting Anthropic Claude, OpenAI GPT, Google Gemini, AWS Bedrock, and many more via the Vercel AI SDK as the core abstraction.

### Core Implementation

**Primary File:** `packages/opencode/src/provider/provider.ts` (1500+ lines)

### Provider Architecture

**1. Provider Schema with Branded Types**

```typescript
// packages/opencode/src/provider/schema.ts
const providerIdSchema = Schema.String.pipe(Schema.brand("ProviderID"))
export type ProviderID = typeof providerIdSchema.Type

export const ProviderID = providerIdSchema.pipe(
  withStatics((schema: typeof providerIdSchema) => ({
    make: (id: string) => schema.makeUnsafe(id),
    zod: z.string().pipe(z.custom<ProviderID>()),
    // Well-known providers
    opencode: schema.makeUnsafe("opencode"),
    anthropic: schema.makeUnsafe("anthropic"),
    openai: schema.makeUnsafe("openai"),
    google: schema.makeUnsafe("google"),
    // ... more
  })),
)
```

**2. Bundled Provider Registry**

```typescript
const BUNDLED_PROVIDERS: Record<string, (options: any) => SDK> = {
  "@ai-sdk/amazon-bedrock": createAmazonBedrock,
  "@ai-sdk/anthropic": createAnthropic,
  "@ai-sdk/azure": createAzure,
  "@ai-sdk/google": createGoogleGenerativeAI,
  "@ai-sdk/google-vertex": createVertex,
  "@ai-sdk/google-vertex/anthropic": createVertexAnthropic,
  "@ai-sdk/openai": createOpenAI,
  "@ai-sdk/openai-compatible": createOpenAICompatible,
  "@openrouter/ai-sdk-provider": createOpenRouter,
  "@ai-sdk/xai": createXai,
  "@ai-sdk/mistral": createMistral,
  "@ai-sdk/groq": createGroq,
  "@ai-sdk/deepinfra": createDeepInfra,
  "@ai-sdk/cerebras": createCerebras,
  "@ai-sdk/cohere": createCohere,
  "@ai-sdk/gateway": createGateway,
  "@ai-sdk/togetherai": createTogetherAI,
  "@ai-sdk/perplexity": createPerplexity,
  "@ai-sdk/vercel": createVercel,
  "gitlab-ai-provider": createGitLab,
  "@ai-sdk/github-copilot": createGitHubCopilotOpenAICompatible,
}
```

**3. Custom Loaders for Provider-Specific Logic**

```typescript
const CUSTOM_LOADERS: Record<string, CustomLoader> = {
  async anthropic() {
    return {
      autoload: false,
      options: {
        headers: {
          "anthropic-beta": "interleaved-thinking-2025-05-14,fine-grained-tool-streaming-2025-05-14",
        },
      },
    }
  },
  async "amazon-bedrock"() {
    // Complex region prefixing logic for cross-region inference
    // e.g., "us.claude-3-5-sonnet" for US regions
  },
  "github-copilot": async () => {
    return {
      autoload: false,
      async getModel(sdk, modelID) {
        if (useLanguageModel(sdk)) return sdk.languageModel(modelID)
        return shouldUseCopilotResponsesApi(modelID) ? sdk.responses(modelID) : sdk.chat(modelID)
      },
    }
  },
}
```

### Model Discovery and Caching

**1. Remote Model List from models.dev**

```typescript
export namespace ModelsDev {
  export const Data = lazy(async () => {
    const result = await Filesystem.readJson(Flag.OPENCODE_MODELS_PATH ?? filepath).catch(() => {})
    if (result) return result
    // Fallback to bundled snapshot
    const snapshot = await import("./models-snapshot.js").then((m) => m.snapshot).catch(() => undefined)
    if (snapshot) return snapshot
    // Fetch from remote
    if (Flag.OPENCODE_DISABLE_MODELS_FETCH) return {}
    const json = await fetch(`${url()}/api.json`).then((x) => x.text())
    return JSON.parse(json)
  })
}
```

**2. SDK Caching by Provider/Model Hash**

```typescript
const key = Hash.fast(JSON.stringify({ providerID: model.providerID, npm: model.api.npm, options }))
const existing = s.sdk.get(key)
if (existing) return existing

// Otherwise create new SDK instance
const loaded = bundledFn({ name: model.providerID, ...options })
s.sdk.set(key, loaded)
return loaded
```

### Cross-Region Inference for AWS Bedrock

**Sophisticated region prefixing logic:**

```typescript
switch (regionPrefix) {
  case "us": {
    const modelRequiresPrefix = ["nova-micro", "nova-lite", "nova-pro", "claude", "deepseek"].some(...)
    if (modelRequiresPrefix && !isGovCloud) {
      modelID = `${regionPrefix}.${modelID}`  // e.g., "us.claude-3-5-sonnet"
    }
    break
  }
  case "eu": { /* similar */ }
  case "ap": { /* with Australia and Tokyo special cases */ }
}
```

### Provider-Specific Transformations

**File:** `packages/opencode/src/provider/transform.ts`

Handles provider-specific message normalization:

1. **Anthropic**: Filters empty content, sanitizes tool call IDs
2. **Mistral**: Requires exactly 9-character alphanumeric tool call IDs
3. **Interleaved reasoning**: Handles `reasoning_content` / `reasoning_details` fields

### Error Handling

**File:** `packages/opencode/src/provider/error.ts`

Comprehensive error parsing with overflow detection:

```typescript
const OVERFLOW_PATTERNS = [
  /prompt is too long/i,                    // Anthropic
  /input is too long for requested model/i,  // Amazon Bedrock
  /exceeds the context window/i,             // OpenAI
  /input token count.*exceeds the maximum/i,// Google
  // ... 20+ patterns
]

export function parseAPICallError(input): ParsedAPICallError {
  if (isOverflow(m) || statusCode === 413) {
    return { type: "context_overflow", message: m }
  }
  return { type: "api_error", isRetryable: ..., statusCode: ... }
}
```

---

## Feature 3: Built-in Agent System

### Overview

Three primary modes (build/plan/general) plus specialized subagents (explore, compaction, title, summary) that can be switched via Tab key or automatically selected based on context.

### Core Implementation

**Built-in Agents:** `packages/opencode/src/agent/agent.ts` (lines 105-232)

### Agent Definitions

**1. Build Agent (Primary)**

```typescript
build: {
  name: "build",
  description: "The default agent. Executes tools based on configured permissions.",
  permission: Permission.merge(
    defaults,
    Permission.fromConfig({
      question: "allow",
      plan_enter: "allow",
    }),
    user,
  ),
  mode: "primary",
  native: true,
}
```

**2. Plan Agent (Read-Only Analysis)**

```typescript
plan: {
  name: "plan",
  description: "Plan mode. Disallows all edit tools.",
  permission: Permission.merge(
    defaults,
    Permission.fromConfig({
      question: "allow",
      plan_exit: "allow",
      edit: { "*": "deny", "*.md": "allow" },  // Only markdown edits allowed
    }),
    user,
  ),
  mode: "primary",
  native: true,
}
```

**3. General Agent (Multi-Step Research)**

```typescript
general: {
  name: "general",
  description: `General-purpose agent for researching complex questions and executing multi-step tasks. Use this agent to execute multiple units of work in parallel.`,
  permission: Permission.merge(defaults, Permission.fromConfig({ todowrite: "deny" }), user),
  mode: "subagent",
  native: true,
}
```

**4. Explore Agent (Fast Codebase Search)**

```typescript
explore: {
  name: "explore",
  permission: Permission.merge(
    defaults,
    Permission.fromConfig({
      "*": "deny",
      grep: "allow",
      glob: "allow",
      list: "allow",
      bash: "allow",
      webfetch: "allow",
      websearch: "allow",
      codesearch: "allow",
      read: "allow",
    }),
  ),
  description: `Fast agent specialized for exploring codebases...`,
  mode: "subagent",
  native: true,
}
```

### Agent Prompt Templates

**Location:** `packages/opencode/src/agent/prompt/`

- `compaction.txt` - Context compaction agent
- `explore.txt` - Codebase exploration
- `summary.txt` - Conversation summarization
- `title.txt` - Session titling

### Agent Mode Selection Logic

```typescript
const defaultAgent = Effect.fnUntraced(function* () {
  const c = yield* config()
  if (c.default_agent) {
    const agent = agents[c.default_agent]
    if (!agent) throw new Error(`default agent "${c.default_agent}" not found`)
    if (agent.mode === "subagent") throw new Error(`default agent "${c.default_agent}" is a subagent`)
    if (agent.hidden === true) throw new Error(`default agent "${c.default_agent}" is hidden`)
    return agent.name
  }
  const visible = Object.values(agents).find((a) => a.mode !== "subagent" && a.hidden !== true)
  if (!visible) throw new Error("no primary visible agent found")
  return visible.name
})
```

### Dynamic Agent Customization

Users can override or extend agents via `opencode.json`:

```typescript
for (const [key, value] of Object.entries(cfg.agent ?? {})) {
  if (value.disable) {
    delete agents[key]
    continue
  }
  // Merge custom config with built-in agent
  item.model = Provider.parseModel(value.model)
  item.variant = value.variant ?? item.variant
  item.prompt = value.prompt ?? item.prompt
  item.temperature = value.temperature ?? item.temperature
  item.mode = value.mode ?? item.mode
  item.permission = Permission.merge(item.permission, Permission.fromConfig(value.permission ?? {}))
}
```

### Hidden Agents

Specialized agents hidden from the UI but available for internal use:

- `compaction` - Context compaction (hidden, primary)
- `title` - Generate session titles (hidden, primary)
- `summary` - Summarize conversations (hidden, primary)

---

## Technical Debt and Concerns

### 1. TODO Comments in Provider Code

```typescript
// provider.ts line 133
// @ts-ignore (TODO: kill this code so we dont have to maintain it)
"@ai-sdk/github-copilot": createGitHubCopilotOpenAICompatible,
```

### 2. Direct process.env Access

Several places use `process.env` directly instead of the `Env` abstraction:

```typescript
// provider.ts line 276
const envToken = process.env.AWS_BEARER_TOKEN_BEDROCK
if (envToken) return envToken
```

### 3. Complex Region Prefix Logic

The AWS Bedrock region prefixing is highly complex with many special cases (20+ lines of switch/case logic for 3 regions with different model requirements).

### 4. SDK Fetch Wrapping Complexity

The custom fetch wrapper in `getSDK` handles many edge cases but is complex:

```typescript
options["fetch"] = async (input: any, init?: BunFetchRequestInit) => {
  const fetchFn = customFetch ?? fetch
  // Multiple signal combinations
  // SSE timeout wrapping
  // openai itemId stripping
}
```

---

## Key Files Summary

| File | Purpose | Lines |
|------|---------|-------|
| `packages/opencode/src/agent/agent.ts` | Agent service definition, built-in agents | ~414 |
| `packages/opencode/src/provider/provider.ts` | Provider abstraction, SDK loading | ~1506 |
| `packages/opencode/src/provider/schema.ts` | ProviderID/ModelID branded types | ~39 |
| `packages/opencode/src/provider/models.ts` | Remote model list from models.dev | ~133 |
| `packages/opencode/src/provider/error.ts` | Error parsing and overflow detection | ~195 |
| `packages/opencode/src/provider/transform.ts` | Provider-specific message normalization | ~500+ |
| `packages/opencode/src/agent/prompt/*.txt` | Agent prompt templates | ~4 files |
| `packages/opencode/AGENTS.md` | Agent documentation and style guide | ~129 |

---

## Dependencies Used

- **ai** (Vercel AI SDK) - Core LLM abstraction
- **@ai-sdk/\*** - Provider-specific implementations (20+ packages)
- **effect** - Dependency injection and service composition
- **remeda** - Functional utilities (mergeDeep, sortBy, etc.)
- **zod** - Schema validation
- **fuzzysort** - Fuzzy matching for error suggestions
