# Vercel AI SDK Architecture Analysis

## Architectural Pattern: Layered Provider Abstraction with Adapter Pattern

The Vercel AI SDK follows a **layered architecture** with clear separation of concerns:

```
packages/
  ai/                    # Core SDK - High-level APIs
  provider/              # Interface specifications (contracts)
  provider-utils/        # Shared utilities
  <provider>/           # Concrete provider implementations (adapters)
  <framework>/          # Framework integrations (React, Vue, Svelte, Angular, RSC)
```

### Layer Responsibilities

| Layer | Package | Responsibility |
|-------|---------|----------------|
| **User-Facing API** | `ai` | `generateText`, `streamText`, `generateObject`, etc. |
| **Abstraction** | `provider` | `LanguageModelV4`, `EmbeddingModelV4` interfaces |
| **Utilities** | `provider-utils` | HTTP utilities, schema validation, JSON parsing |
| **Implementation** | `@ai-sdk/<provider>` | Concrete adapters (OpenAI, Anthropic, Google) |
| **Integration** | `<framework>` | React hooks, Vue composables, Svelte stores |

---

## Provider Interface Architecture

### Core Interface: LanguageModelV4

Located in `packages/provider/src/language-model/v4/language-model-v4.ts`:

```typescript
export type LanguageModelV4 = {
  readonly specificationVersion: 'v4';
  readonly provider: string;
  readonly modelId: string;
  supportedUrls: PromiseLike<Record<string, RegExp[]>> | Record<string, RegExp[]>;
  doGenerate(options: LanguageModelV4CallOptions): PromiseLike<LanguageModelV4GenerateResult>;
  doStream(options: LanguageModelV4CallOptions): PromiseLike<LanguageModelV4StreamResult>;
};
```

**Key design decisions:**
- `do` prefix prevents accidental direct usage
- Two methods: `doGenerate` (non-streaming) and `doStream` (streaming)
- Standardized options and result types decouple from provider-specific APIs

### Provider Interface: ProviderV4

Located in `packages/provider/src/provider/v4/provider-v4.ts`:

```typescript
export interface ProviderV4 {
  readonly specificationVersion: 'v4';
  languageModel(modelId: string): LanguageModelV4;
  embeddingModel(modelId: string): EmbeddingModelV4;
  imageModel(modelId: string): ImageModelV4;
  transcriptionModel?(modelId: string): TranscriptionModelV4;
  speechModel?(modelId: string): SpeechModelV4;
  rerankingModel?(modelId: string): RerankingModelV4;
}
```

### Interface Versioning Strategy

Three interface versions exist simultaneously (`v2`, `v3`, `v4`) for backwards compatibility. The SDK uses adapter functions in `packages/ai/src/model/` to convert older versions to v4:

- `asLanguageModelV4()` - converts v2/v3 to v4
- `asEmbeddingModelV4()` - converts embedding models to v4
- `asProviderV4()` - converts providers to v4

---

## Data Flow: generateText to Provider

### Complete Flow Diagram

```
User Code
    │
    ▼
generateText({ model: 'gpt-4o', messages: [...] })
    │
    ▼
resolveLanguageModel(model)
    │  model can be:
    │  1. String ('gpt-4o') -> global provider creates LanguageModelV4
    │  2. LanguageModelV4 instance -> used directly
    │  3. LanguageModelV3/V2 -> converted to V4
    │
    ▼
Standardize Prompt
    │  Converts user-facing prompt to internal Prompt type
    │  packages/ai/src/prompt/standardize-prompt.ts
    │
    ▼
Convert to LanguageModel Prompt
    │  Converts Prompt to LanguageModelV4Prompt
    │  packages/ai/src/prompt/convert-to-language-model-prompt.ts
    │
    ▼
Prepare Call Settings
    │  Merges user settings with defaults
    │  packages/ai/src/prompt/prepare-call-settings.ts
    │
    ▼
Prepare Tools
    │  Converts tools to provider-specific format
    │  packages/ai/src/prompt/prepare-tools-and-tool-choice.ts
    │
    ▼
LOOP: Model Invocation (potentially multiple steps for tool calls)
    │
    ├─▶ Model.doGenerate() or Model.doStream()
    │       │
    │       ▼
    │   Provider Adapter (e.g., OpenAIChatLanguageModel)
    │       │
    │       ▼
    │   Provider API (e.g., OpenAI Chat API)
    │
    ▼
Handle Tool Calls (if any)
    │  Execute tools, feed results back to model
    │  packages/ai/src/generate-text/execute-tool-call.ts
    │
    ▼
Return GenerateTextResult
```

### Key Files in generateText Flow

| File | Purpose |
|------|---------|
| `packages/ai/src/generate-text/generate-text.ts` | Main orchestration |
| `packages/ai/src/model/resolve-model.ts` | Converts model string/object to LanguageModelV4 |
| `packages/ai/src/prompt/standardize-prompt.ts` | Normalizes user prompts |
| `packages/ai/src/prompt/convert-to-language-model-prompt.ts` | Converts to provider-agnostic format |
| `packages/ai/src/prompt/prepare-call-settings.ts` | Merges settings |
| `packages/ai/src/prompt/prepare-tools-and-tool-choice.ts` | Tool preparation |

---

## Concrete Provider Implementation: OpenAI

### Provider Creation Pattern

Located in `packages/openai/src/openai-provider.ts`:

```typescript
export function createOpenAI(options: OpenAIProviderSettings = {}): OpenAIProvider {
  // Configure base URL, headers, API key
  const getHeaders = () => withUserAgentSuffix({ Authorization: `Bearer ${loadApiKey(...)}` }, ...);

  // Factory functions for each model type
  const createChatModel = (modelId) => new OpenAIChatLanguageModel(modelId, { provider, url, headers, fetch });

  const provider = function(modelId) { return createChatModel(modelId); };

  // Attach methods
  provider.specificationVersion = 'v4';
  provider.languageModel = createLanguageModel;
  provider.chat = createChatModel;
  provider.completion = createCompletionModel;
  // ...

  return provider as OpenAIProvider;
}

export const openai = createOpenAI();
```

### Model Adapter Pattern

Located in `packages/openai/src/chat/openai-chat-language-model.ts`:

```typescript
export class OpenAIChatLanguageModel implements LanguageModelV4 {
  readonly specificationVersion = 'v4';
  readonly modelId: OpenAIChatModelId;
  readonly supportedUrls = { 'image/*': [/^https?:\/\/.*$/] };

  private readonly config: OpenAIChatConfig;

  async doGenerate(options: LanguageModelV4CallOptions) {
    // 1. Parse provider options
    const openaiOptions = parseProviderOptions({ provider: 'openai', providerOptions, schema: ... });

    // 2. Convert prompt to OpenAI format
    const { messages } = convertToOpenAIChatMessages({ prompt: options.prompt, ... });

    // 3. Make API call
    const response = await postJsonToApi({ url: this.config.url(...), headers, body: {...} });

    // 4. Convert response to standard format
    return {
      content: response.choices.map(choice => ({ type: 'text', text: choice.message.content })),
      finishReason: mapOpenAIFinishReason(response.choices[0].finish_reason),
      usage: convertOpenAIChatUsage(response.usage),
      warnings: [],
    };
  }

  doStream(options) { /* Similar pattern with streaming */ }
}
```

---

## Framework Integrations

### React Integration

**Location:** `packages/react/src/`

| File | Purpose |
|------|---------|
| `use-chat.ts` | Hook for chat UIs |
| `use-completion.ts` | Hook for completion UIs |
| `use-object.ts` | Hook for structured object generation |
| `chat.react.ts` | Core Chat class with state management |

**Architecture:**
- `useChat` wraps `Chat` class (in `chat.react.ts`)
- `Chat` extends `AbstractChat` from `ai` package
- Framework-agnostic core (`AbstractChat`) with React-specific state (`ReactChatState`)

```typescript
// Simplified useChat flow
export function useChat(options) {
  const chatRef = useRef(new Chat(options));  // Core Chat instance
  const [state, setState] = useSyncExternalStore(
    chatRef.current.subscribe,  // React -> Core state sync
    chatRef.current.getSnapshot
  );
  return { ...state, sendMessage: chatRef.current.sendMessage };
}
```

### RSC (React Server Components) Integration

**Location:** `packages/rsc/src/`

| File | Purpose |
|------|---------|
| `rsc-server.ts` | Server-side: `streamUI`, `createAI` |
| `rsc-client.ts` | Client-side: `useStreamableValue`, `useAIState` |
| `streamable-ui/` | UI streaming utilities |
| `streamable-value/` | Value streaming utilities |

**Key concepts:**
- `createStreamableUI()` - Creates streamable UI components
- `createAI()` - Creates AI state management for RSC
- Server actions for client-server communication

### Other Frameworks

| Package | Entry Point | Pattern |
|---------|-------------|---------|
| `packages/vue/src/` | `use-chat.ts`, `use-completion.ts` | Vue composables |
| `packages/svelte/src/` | `chat.svelte.ts`, `completion.svelte.ts` | Svelte stores |
| `packages/angular/src/` | Angular services | Angular DI pattern |

---

## Middleware System

**Location:** `packages/ai/src/middleware/`

Enables transformation of model calls without modifying core logic:

```typescript
// wrap-language-model.ts
export function wrapLanguageModel({ model, middleware }) {
  return {
    specificationVersion: 'v4',
    provider: model.provider,
    modelId: model.modelId,
    supportedUrls: model.supportedUrls,

    async doGenerate(options) {
      const transformedOptions = middleware.transformParams?.({ model, params: options }) ?? options;
      return middleware.wrapGenerate?.({
        model,
        params: transformedOptions,
        fn: () => model.doGenerate(transformedOptions)
      }) ?? model.doGenerate(transformedOptions);
    },

    doStream(options) { /* Similar */ }
  };
}
```

**Available middlewares:**
- `add-tool-input-examples-middleware.ts`
- `default-settings-middleware.ts`
- `extract-json-middleware.ts`
- `extract-reasoning-middleware.ts`
- `simulate-streaming-middleware.ts`

---

## Key Abstractions and Base Classes

### Provider Abstraction Layers

```
LanguageModel (union type)
    │
    ├─▶ LanguageModelV4
    │       │
    │       └─▶ Concrete implementations (OpenAIChatLanguageModel, etc.)
    │
    ├─▶ LanguageModelV3 (adapter -> V4)
    └─▶ LanguageModelV2 (adapter -> V4)
```

### Model Type Hierarchy

```
packages/provider/src/
    language-model/
        v4/
            language-model-v4.ts          # Core interface
            language-model-v4-call-options.ts
            language-model-v4-generate-result.ts
            language-model-v4-stream-result.ts
            language-model-v4-content.ts
            language-model-v4-function-tool.ts
            ...
```

### Error Handling Pattern

All errors extend `AISDKError` with marker pattern:

```typescript
export class NoSuchModelError extends AISDKError {
  private readonly [symbol] = true;
  static isInstance(error: unknown): boolean {
    return AISDKError.hasMarker(error, marker);
  }
}
```

---

## Module Map

### Core SDK (`packages/ai/src/`)

| Module | Files | Purpose |
|--------|-------|---------|
| `generate-text/` | `generate-text.ts`, `stream-text.ts` | Text generation orchestration |
| `generate-object/` | `generate-object.ts`, `stream-object.ts` | Structured output generation |
| `prompt/` | `standardize-prompt.ts`, `convert-to-language-model-prompt.ts` | Prompt normalization |
| `model/` | `resolve-model.ts`, `as-*-v4.ts` | Model resolution and adaptation |
| `middleware/` | `wrap-language-model.ts`, etc. | Middleware system |
| `types/` | `language-model.ts`, `usage.ts` | Core type definitions |
| `ui/` | Chat UI components and utilities | UI primitives |
| `error/` | Error classes | Standardized errors |

### Provider Package (`packages/provider/src/`)

| Module | Purpose |
|--------|---------|
| `language-model/v4/` | Language model interface specifications |
| `embedding-model/v4/` | Embedding model interface |
| `image-model/v4/` | Image generation interface |
| `provider/v4/` | Provider factory interface |
| `shared/` | Shared types (warnings, headers, metadata) |

### Provider Utils (`packages/provider-utils/src/`)

| Module | Purpose |
|--------|---------|
| HTTP utilities | `post-to-api.ts`, `get-from-api.ts` |
| Schema utilities | `schema.ts`, `parse-json.ts` |
| Authentication | `load-api-key.ts`, `with-user-agent-suffix.ts` |
| Stream parsing | `parse-json-event-stream.ts` |
| Tool factories | `provider-tool-factory.ts` |

---

## Dependency Graph

```
                    ┌─────────────────────────────────────────┐
                    │              packages/ai                 │
                    │  (generateText, streamText, etc.)         │
                    └───────────────────┬─────────────────────┘
                                        │
                    ┌───────────────────┴─────────────────────┐
                    │                                         │
                    ▼                                         ▼
    ┌───────────────────────────┐         ┌───────────────────────────┐
    │   @ai-sdk/provider-utils   │         │    @ai-sdk/provider       │
    │  (HTTP, schemas, auth)     │         │  (LanguageModelV4, etc.)  │
    └───────────────────────────┘         └───────────────────────────┘
                    ▲                                         ▲
                    │                                         │
    ┌───────────────┴────────────────────────────────────────┴───────┐
    │                                                                │
    ▼                                                                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│@ai-sdk/openai│  │@ai-sdk/anthropic│ │  @ai-sdk/google │  │  @ai-sdk/azure  │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

---

## Summary

The Vercel AI SDK architecture demonstrates several key patterns:

1. **Adapter Pattern**: Each provider (OpenAI, Anthropic, etc.) implements the `LanguageModelV4` interface, allowing the core SDK to work with any provider without modification.

2. **Layered Architecture**: Clear separation between user-facing APIs (`ai`), interfaces (`provider`), utilities (`provider-utils`), and implementations (`@ai-sdk/<provider>`).

3. **Interface Versioning**: Multiple interface versions coexist with adapter functions converting older versions to the current standard.

4. **Framework Abstraction**: Framework-specific packages (React, Vue, Svelte) wrap a framework-agnostic core, enabling consistent APIs across different UI frameworks.

5. **Middleware System**: Allows cross-cutting concerns (logging, caching, defaults) to be applied without modifying core logic.

6. **Prompt Standardization**: User-facing prompts are normalized through multiple transformation stages before reaching the provider.

This architecture enables:
- Easy addition of new providers (implement `LanguageModelV4`)
- Easy addition of new framework integrations (wrap the core)
- Consistent API across different AI providers
- Flexible customization via middleware
