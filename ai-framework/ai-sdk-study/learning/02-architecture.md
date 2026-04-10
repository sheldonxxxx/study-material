# Vercel AI SDK Architecture

## Overview

The Vercel AI SDK is a TypeScript monorepo providing a unified interface for AI providers (OpenAI, Anthropic, Google, etc.) with framework integrations (React, Vue, Svelte, Angular).

**Repository:** `https://github.com/vercel/ai`
**Type:** Monorepo (pnpm workspaces + Turborepo)
**Packages:** 55 packages (core, providers, frameworks)

---

## Layered Architecture

```
packages/
  ai/                    # Core SDK - High-level APIs
  provider/              # Interface specifications (contracts)
  provider-utils/        # Shared utilities
  <provider>/           # Concrete provider implementations
  <framework>/          # Framework integrations
```

### Layer Responsibilities

| Layer | Package | Responsibility |
|-------|---------|----------------|
| **User-Facing API** | `ai` | `generateText`, `streamText`, `generateObject`, `embed`, etc. |
| **Abstraction** | `provider` | `LanguageModelV4`, `EmbeddingModelV4`, `ImageModelV4` interfaces |
| **Utilities** | `provider-utils` | HTTP utilities, schema validation, JSON parsing, retry logic |
| **Implementation** | `@ai-sdk/<provider>` | Concrete adapters (OpenAI, Anthropic, Google) |
| **Integration** | `<framework>` | React hooks, Vue composables, Svelte stores, Angular services |

### Dependency Graph

```
                    packages/ai
                  (generateText, etc.)
                        |
        +---------------+---------------+
        |                               |
packages/provider-utils          packages/provider
  (HTTP, schemas, auth)          (LanguageModelV4, etc.)
        ^                               ^
        |                               |
        +-------------------------------+
                        |
    ┌-------+-------+-------+-------+
    |       |       |       |       |
 @ai-sdk/ @ai-sdk/ @ai-sdk/ @ai-sdk/
  openai anthropic google  azure
```

---

## Module Responsibilities

### Core SDK (`packages/ai/src/`)

| Module | Files | Purpose |
|--------|-------|---------|
| `generate-text/` | `generate-text.ts`, `stream-text.ts` | Text generation with step loop |
| `generate-object/` | `generate-object.ts`, `stream-object.ts` | Structured output with Zod |
| `embed/` | `embed.ts`, `embed-many.ts` | Vector embeddings |
| `generate-image/` | `generate-image.ts` | Image generation |
| `generate-speech/` | `generate-speech.ts` | Text-to-speech |
| `generate-video/` | `generate-video.ts` | Video generation |
| `prompt/` | `standardize-prompt.ts`, `convert-to-*.ts` | Prompt normalization |
| `model/` | `resolve-model.ts`, `as-*-v4.ts` | Model resolution and adaptation |
| `middleware/` | `wrap-language-model.ts` | Middleware composition |
| `ui/` | `chat.ts`, `completion.ts`, `object.ts` | AbstractChat for framework hooks |
| `error/` | Error classes | Standardized error hierarchy |
| `agent/` | `tool-loop-agent.ts`, `agent.ts` | Agent framework |

### Provider Package (`packages/provider/src/`)

| Module | Purpose |
|--------|---------|
| `language-model/v4/` | Language model interface (`LanguageModelV4`) |
| `embedding-model/v4/` | Embedding model interface |
| `image-model/v4/` | Image generation interface |
| `speech-model/v4/` | Text-to-speech interface |
| `transcription-model/v4/` | Speech-to-text interface |
| `reranking-model/v4/` | Search reranking interface |
| `provider/v4/` | Provider factory interface (`ProviderV4`) |
| `shared/` | Shared types (warnings, headers, metadata) |

### Provider Utils (`packages/provider-utils/src/`)

| Utility | Purpose |
|---------|---------|
| `postJsonToApi`, `postFormDataToApi` | HTTP requests with error handling |
| `createJsonResponseHandler` | Standardized JSON response parsing |
| `loadApiKey` | API key loading from options/env |
| `combineHeaders`, `withUserAgentSuffix` | Header manipulation |
| `retry`, `prepareRetries` | Exponential backoff retry |
| `safeParseJSON`, `parsePartialJson` | JSON parsing |
| `tool` | Tool definition helper |
| `DelayedPromise` | Lazy promise resolution |

---

## Communication Patterns

### 1. generateText to Provider Flow

```
User Code
    │
    ▼
generateText({ model, messages, ... })
    │
    ▼
resolveLanguageModel(model)  ──► LanguageModelV4 instance
    │
    ▼
standardizePrompt({ system, prompt, messages })
    │  Normalizes to { system?, messages: ModelMessage[] }
    ▼
prepareCallSettings(settings)
    │  Merges user settings with defaults
    ▼
LOOP: stepModel.doGenerate() + execute tools
    │
    ├─► Model.doGenerate() or Model.doStream()
    │       │
    │       ▼
    │   Provider Adapter (e.g., OpenAIChatLanguageModel)
    │       │
    │       ▼
    │   Provider API (OpenAI Chat API)
    │
    ▼
Handle Tool Calls (if any)
    │  Execute tools, feed results back to model
    ▼
Return GenerateTextResult
```

**Key files:**
- `packages/ai/src/generate-text/generate-text.ts` (1420 lines)
- `packages/ai/src/model/resolve-model.ts`
- `packages/ai/src/prompt/standardize-prompt.ts`

### 2. Framework Hook Communication

React, Vue, Svelte, and Angular all wrap `AbstractChat` with framework-specific state:

```
AbstractChat (core in packages/ai/src/ui/chat.ts)
    │
    +-- ReactChatState (useSyncExternalStore)
    +-- VueChatState (ref/reactive)
    +-- SvelteChatState ($state runes)
    +-- AngularChatState (signals)
```

**Pattern:**
```typescript
// React
const chatRef = useRef(new Chat(options));
const messages = useSyncExternalStore(
  chatRef.current.subscribe,
  () => chatRef.current.messages,
);

// Svelte
class SvelteChatState {
  messages = $state<UI_MESSAGE[]>([]);
}
```

### 3. Middleware Pipeline

Middleware transforms model calls without modifying core logic:

```typescript
// wrap-language-model.ts
export function wrapLanguageModel({ model, middleware }) {
  return [...asArray(middlewareArg)]
    .reverse()
    .reduce((wrappedModel, middleware) =>
      doWrap({ model: wrappedModel, middleware }), model);
}
```

**Middleware interface:**
- `transformParams` - Modify request before calling model
- `wrapGenerate` - Wrap non-streaming call
- `wrapStream` - Wrap streaming call

---

## Key Architectural Decisions

### 1. Interface Versioning

Multiple interface versions coexist (`v2`, `v3`, `v4`) with adapter functions converting older to newer:

- `asLanguageModelV4()` - converts v2/v3 to v4
- `asEmbeddingModelV4()` - converts embedding models to v4
- `asProviderV4()` - converts providers to v4

**File:** `packages/ai/src/model/as-language-model-v4.ts`

### 2. Provider Adapter Pattern

Each provider implements `LanguageModelV4` interface:

```typescript
// packages/openai/src/chat/openai-chat-language-model.ts
export class OpenAIChatLanguageModel implements LanguageModelV4 {
  readonly specificationVersion = 'v4';
  readonly modelId: OpenAIChatModelId;

  async doGenerate(options: LanguageModelV4CallOptions) {
    // 1. Parse provider options
    // 2. Convert prompt to OpenAI format
    // 3. Make API call
    // 4. Convert response to standard format
    return { content, finishReason, usage, warnings };
  }
}
```

### 3. Step Loop for Tool Execution

`generateText` loops until stop conditions are met:

```typescript
// packages/ai/src/generate-text/generate-text.ts
do {
  const response = await stepModel.doGenerate({ prompt, tools, ... });
  const toolCalls = extractToolCalls(response.content);

  if (toolCalls.length > 0) {
    const results = await executeTools({ toolCalls });
    addResultsToMessages(results);
  }
} while (hasToolCalls && !isStopConditionMet);
```

### 4. Output Strategy Pattern

Structured object generation uses strategy pattern:

```typescript
// packages/ai/src/generate-object/output-strategy.ts
interface OutputStrategy<PARTIAL, RESULT, ELEMENT_STREAM> {
  readonly type: 'object' | 'array' | 'enum' | 'no-schema';
  jsonSchema(): Promise<JSONSchema7 | undefined>;
  validatePartialResult({ value, textDelta }): Promise<ValidationResult<...>>;
  validateFinalResult(value, context): Promise<ValidationResult<RESULT>>;
}
```

Strategies:
- `objectOutputStrategy` - Single object with Zod schema
- `arrayOutputStrategy` - Array wrapped in `{ elements: [] }`
- `enumOutputStrategy` - Enum wrapped in `{ result: enum }`
- `noSchemaOutputStrategy` - Raw JSON passthrough

### 5. Framework Abstraction

Framework packages wrap framework-agnostic core:

```
packages/ai/src/ui/chat.ts (AbstractChat)
        │
        ├── packages/react/src/chat.react.ts (ReactChatState)
        ├── packages/vue/src/chat.vue.ts (VueChatState)
        ├── packages/svelte/src/chat.svelte.ts (SvelteChatState)
        └── packages/angular/src/lib/chat.ng.ts (AngularChatState)
```

### 6. Error Handling Hierarchy

All errors extend `AISDKError` with marker pattern:

```typescript
// packages/ai/src/error/ai-sdk-error.ts
export class AISDKError extends Error {
  static isInstance(error: unknown): boolean;
  static hasMarker(error: unknown, marker: symbol): boolean;
}

export class NoSuchModelError extends AISDKError {
  private readonly [symbol] = true;
  static isInstance(error: unknown): boolean;
}
```

---

## Data Flow Examples

### generateText with Tools

```
1. User: generateText({ model: 'gpt-4o', messages, tools: [weatherTool] })
2. resolveLanguageModel('gpt-4o') -> OpenAIChatLanguageModel instance
3. standardizePrompt({ messages }) -> { messages: [...] }
4. Loop (step 1):
   a. model.doGenerate({ prompt, tools }) -> { content: [tool-call] }
   b. Extract tool calls: [ { name: 'weather', args: { location: 'NYC' } } ]
   c. Execute weatherTool.execute({ location: 'NYC' }) -> 'sunny'
   d. Add tool result to messages
5. Loop (step 2):
   a. model.doGenerate({ prompt, tools, toolResults }) -> { content: [text] }
   b. No tool calls, exit loop
6. Return { text: 'The weather in NYC is sunny', ... }
```

### streamObject with Zod

```
1. User: streamObject({ model, schema: z.object({name, age}), prompt })
2. getOutputStrategy(schema) -> objectOutputStrategy
3. model.doStream({ responseFormat: { json: schema }, prompt })
4. For each streaming chunk:
   a. Parse partial JSON: parsePartialJson(accumulatedText)
   b. Validate: outputStrategy.validatePartialResult(value)
   c. Emit: controller.enqueue({ type: 'object', object })
5. Final validation: outputStrategy.validateFinalResult(value)
6. Return: { object: { name: 'John', age: 30 }, ... }
```

---

## Framework Integration Details

### React (`packages/react/`)

| File | Purpose |
|------|---------|
| `use-chat.ts` | Main hook using `useSyncExternalStore` |
| `use-completion.ts` | Completion hook using SWR |
| `use-object.ts` | Object hook with `parsePartialJson` |
| `chat.react.ts` | `ReactChatState` implementation |

**Stale closure pattern:**
```typescript
const callbacksRef = useRef(!('chat' in options) ? { onToolCall, onData, ... } : {});
```

### Vue (`packages/vue/`)

Vue requires explicit cloning due to deep reactivity quirks:
```typescript
replaceMessage = (index: number, message: UI_MESSAGE) => {
  this.messagesRef.value[index] = { ...message };
};
```

### Svelte (`packages/svelte/`)

Uses `$state` runes for fine-grained reactivity:
```typescript
messages = $state<UI_MESSAGE[]>([]);
snapshot = <T>(thing: T): T => $state.snapshot(thing);
```

### Angular (`packages/angular/`)

Uses signals with `structuredClone` for snapshots:
```typescript
readonly #messages = signal<UI_MESSAGE[]>([]);
snapshot = <T>(thing: T): T => structuredClone(thing);
```

---

## Summary

The Vercel AI SDK architecture demonstrates:

1. **Layered Provider Abstraction** - Clear separation between user-facing APIs, interfaces, utilities, and implementations

2. **Adapter Pattern** - Each provider implements `LanguageModelV4`, enabling pluggable providers

3. **Interface Versioning** - Multiple versions coexist with adapter functions for backwards compatibility

4. **Step Loop** - Tool execution via iterative model calls with message accumulation

5. **Output Strategy** - Strategy pattern for different structured output modes

6. **Framework Abstraction** - `AbstractChat` + `ChatState` interface enables consistent APIs across React, Vue, Svelte, Angular

7. **Middleware System** - Cross-cutting concerns via transform/wrap hooks

8. **Error Hierarchy** - Marker-based error identification for precise error handling
