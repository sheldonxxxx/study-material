# Vercel AI SDK Feature Deep Dive

## Overview

This document synthesizes research from four feature batches into a unified technical deep-dive of the Vercel AI SDK. Features are ordered by priority (CORE first, then SECONDARY), revealing a significant maturity gradient across the feature set.

**Key Insight:** The SDK exhibits a clear three-tier maturity model:
1. **Tier 1 (Most Mature):** Text generation, structured objects, embeddings -- full telemetry, callbacks, and comprehensive test coverage
2. **Tier 2 (Moderate):** Image generation, UI hooks, provider architecture -- good infrastructure, some features still experimental
3. **Tier 3 (Least Mature):** Speech, video, transcription -- missing telemetry/callbacks, limited providers, `experimental_` prefixes

---

## CORE FEATURES

## 1. Text Generation

**Priority:** CORE | **Files:** 54 subdirectories | **Providers:** 40+

### Architecture

Text generation (`generateText`, `streamText`) is the SDK's most mature feature, built on a **step loop pattern** that supports multi-turn tool calling and streaming.

```
User Code
    │
    ▼
generateText()
    │
    ▼
1. resolveLanguageModel() ──────────────────────► Model
    │
2. standardizePrompt() ──► { system?, messages }
    │
3. prepareCallSettings()
    │
4. retry() wrapper
    │
5. stepModel.doGenerate({ prompt, tools, ... })
    │                                              ┌───────────────────────┐
    └──────────────────────────────────────────────► LanguageModelV4       │
                                                        (Provider Impl)    │
                                                    └───────────────────────┘
```

### Step Loop Pattern

The step loop enables multi-turn interactions with automatic tool execution:

```typescript
do {
  // 1. Optional prepareStep() callback
  const prepareStepResult = await prepareStep?.({ model, steps, stepNumber, messages });

  // 2. Convert prompt to provider format
  const promptMessages = await convertToLanguageModelPrompt({ prompt, supportedUrls, download });

  // 3. Call language model
  currentModelResponse = await retry(() =>
    stepModel.doGenerate({ tools: stepTools, toolChoice: stepToolChoice, prompt, ... })
  );

  // 4. Parse and execute tool calls
  const stepToolCalls = await Promise.all(
    currentModelResponse.content
      .filter((part): part is LanguageModelV4ToolCall => part.type === 'tool-call')
      .map(toolCall => parseToolCall({ toolCall, tools, repairToolCall, system, messages }))
  );

  // 5. Execute tools and add results to messages
  if (tools != null) {
    clientToolOutputs.push(...await executeTools({ toolCalls: clientToolCalls, ... }));
  }

  // 6. Check stop condition
} while (hasClientToolCalls && !isStopConditionMet({ stopConditions, steps }));
```

**Default stop condition:** `stepCountIs(1)` (single step, no tools).

### Key Implementation Details

**Tool Call Handling:** Tools use the `tool()` helper with Zod schema validation and optional streaming support:

```typescript
const weatherTool = tool({
  name: 'weather',
  description: 'Get weather for a location',
  parameters: z.object({ location: z.string() }),
  execute: async ({ location }) => `The weather in ${location} is sunny.`,
  stream: async function* ({ location }) {
    yield { type: 'preliminary', output: `Looking up ${location}...` };
    yield { type: 'output', output: `Sunny in ${location}` };
  }
});
```

**Retry Logic:** Uses `prepareRetries()` with configurable `maxRetries` and `abortSignal` support.

**Memory Management:** `experimental_include` option controls retention of large payloads (images, request bodies).

### Clever Solutions

1. **DelayedPromise:** Lazy promise resolution for streaming results
2. **StitchableStream:** Combines multiple streams into one
3. **Tool Call Repair:** Invalid tool calls can be automatically repaired

### Technical Debt

- TODO at line 842: "AI SDK 6: invalid inputs should not require output parts"
- Complex merge pattern for tool approval handling
- Deferred tool results need special handling for multi-turn scenarios

---

## 2. Structured Object Generation

**Priority:** CORE | **Files:** ~6 core files | **Providers:** Via LanguageModelV4

### Output Strategy Pattern

Object generation uses a strategy pattern supporting four modes:

```typescript
interface OutputStrategy<PARTIAL, RESULT, ELEMENT_STREAM> {
  readonly type: 'object' | 'array' | 'enum' | 'no-schema';

  jsonSchema(): Promise<JSONSchema7 | undefined>;
  validatePartialResult({ value, textDelta, ... }): Promise<ValidationResult<...>>;
  validateFinalResult(value, context): Promise<ValidationResult<RESULT>>;
  createElementStream(originalStream): ELEMENT_STREAM;
}
```

### Array Handling

Arrays are wrapped in an object structure since most LLMs don't generate bare arrays:

```typescript
jsonSchema: async () => ({
  type: 'object',
  properties: {
    elements: { type: 'array', items: itemSchema }  // LLM generates { elements: [...] }
  },
  required: ['elements'],
})
```

### Streaming with Partial Validation

`streamObject` uses `parsePartialJson` for incremental parsing:

```typescript
const { value: currentObjectJson, state: parseState } = await parsePartialJson(accumulatedText);

if (validationResult.success) {
  latestObject = validationResult.value.partial;
  controller.enqueue({ type: 'object', object: latestObject });
}
```

### Zod Integration

The SDK maintains Zod 3/4 compatibility via conditional imports and uses `safeValidateTypes` for validation.

---

## 3. Agent Framework

**Priority:** CORE | **Files:** ~10 files | **Providers:** Via generateText/streamText

### ToolLoopAgent Architecture

`ToolLoopAgent` is a **thin wrapper** around `generateText`/`streamText`:

```typescript
class ToolLoopAgent implements Agent {
  async generate({ ...options }) {
    return generateText({
      ...(await this.prepareCall(options)),
      stopWhen: this.settings.stopWhen ?? stepCountIs(20), // Default 20 steps
    });
  }
}
```

### Middleware Framework

The middleware system uses a **reverse application pattern** for composable transformations:

```typescript
export const wrapLanguageModel = ({ model, middleware }) => {
  return [...asArray(middlewareArg)]
    .reverse()  // Middleware applied in reverse order
    .reduce((wrappedModel, middleware) =>
      doWrap({ model: wrappedModel, middleware }), model);
};
```

**Middleware hooks:**
- `transformParams` -- Transform parameters before model call
- `wrapGenerate` / `wrapStream` -- Wrap operations
- `overrideProvider` / `overrideModelId` -- Override metadata

**Built-in middleware:**
- `extractJsonMiddleware` -- Extracts JSON from markdown blocks
- `simulateStreamingMiddleware` -- Makes non-streaming appear as streaming
- `addToolInputExamplesMiddleware` -- Adds examples to tool prompts

### Tool Lifecycle

```typescript
// 1. Notify start
await notify({ event: { ... }, callbacks: onToolCallStart });

// 2. Execute with timeout
const toolAbortSignal = AbortSignal.timeout(toolTimeoutMs);
output = await executeTool({ execute: tool.execute.bind(tool), input, options });

// 3. Handle streaming results
for await (const part of stream) {
  if (part.type === 'preliminary') {
    onPreliminaryToolResult?.({ output: part.output, preliminary: true });
  }
}

// 4. Notify finish
await notify({ event: { ..., success: true, output, durationMs }, callbacks: onToolCallFinish });
```

---

## 4. UI Integration / Chat Hooks

**Priority:** CORE | **Framework Packages:** React, Vue, Svelte, Angular, RSC

### Architecture Pattern

The chat hooks use **shared core + framework-specific state**:

```
AbstractChat (packages/ai/src/ui/chat.ts)
    │
    ├── ReactChatState (useSyncExternalStore)
    ├── VueChatState (ref/reactive)
    ├── SvelteChatState ($state runes)
    └── AngularChatState (signals)
```

### AbstractChat Core

The framework-agnostic core contains all logic:

```typescript
export abstract class AbstractChat<UI_MESSAGE extends UIMessage> {
  readonly id: string;
  protected state: ChatState<UI_MESSAGE>;
  private transport: ChatTransport<UI_MESSAGE>;
  private jobExecutor = new SerialJobExecutor();
  private activeResponse: ActiveResponse<UI_MESSAGE> | undefined;
}
```

**Key design decisions:**

1. **ChatState interface** -- Defines state management contract
2. **SerialJobExecutor** -- Ensures sequential tool execution (prevents race conditions)
3. **Transport abstraction** -- Allows customization of API communication
4. **Status state machine:** `'submitted'` -> `'streaming'` -> `'ready'` | `'error'`

### Framework-Specific Patterns

**React:** Uses `useSyncExternalStore` with `callbacksRef` pattern to avoid stale closures:

```typescript
const callbacksRef = useRef(!('chat' in options) ? { onToolCall, onData, ... } : {});
// Callbacks updated every render but Chat instance stays stable
```

**Vue:** Requires explicit cloning in `replaceMessage` due to deep reactivity quirks:

```typescript
replaceMessage = (index: number, message: UI_MESSAGE) => {
  this.messagesRef.value[index] = { ...message }; // Clone to trigger updates
};
```

**Svelte:** Uses `$state` runes for fine-grained reactivity.

**Angular:** Uses signals with structured cloning for snapshots.

### useCompletion and useObject

- `useCompletion` uses SWR for shared state across components
- `useObject` streams JSON incrementally via `parsePartialJson`

---

## 5. Unified Provider Architecture

**Priority:** CORE | **Providers:** 33+ provider packages | **Stack:** 4-layer architecture

### Layered Architecture

```
packages/ai (Core SDK)
       │
packages/provider (Interface Specifications)
       │
packages/provider-utils (Shared Utilities)
       │
packages/<provider-*> (33 Provider Implementations)
```

### Versioned Interface Pattern

Each feature has versioned interfaces (v2, v3, v4) with a `specificationVersion` field:

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

**Key patterns:**
- `do` prefix prevents accidental direct usage
- Versioned specs enable gradual migration
- `specificationVersion` field declares implemented version

### Provider Interface Catalog

| Interface | Purpose |
|-----------|---------|
| `LanguageModelV4` | Text generation |
| `ImageModelV4` | Image generation |
| `EmbeddingModelV4` | Vector embeddings |
| `SpeechModelV4` | Text-to-speech |
| `TranscriptionModelV4` | Speech-to-text |
| `RerankingModelV4` | Search reranking |

### ProviderV4 Interface

```typescript
export interface ProviderV4 {
  readonly specificationVersion: 'v4';
  languageModel(modelId: string): LanguageModelV4;
  embeddingModel(modelId: string): EmbeddingModelV4;
  imageModel(modelId: string): ImageModelV4;
  transcriptionModel?(modelId: string): TranscriptionModelV4;  // Optional
  speechModel?(modelId: string): SpeechModelV4;                  // Optional
  rerankingModel?(modelId: string): RerankingModelV4;           // Optional
}
```

### Provider Utils

Shared utilities in `@ai-sdk/provider-utils`:

| Utility | Purpose |
|---------|---------|
| `postJsonToApi` | JSON requests with error handling |
| `postFormDataToApi` | Form data for image uploads |
| `createJsonResponseHandler` | Standardized JSON parsing |
| `loadApiKey` | Load API key from options or environment |
| `retry` | Automatic retry with exponential backoff |
| `safeParseJSON` | Secure JSON parsing |

### Error Handling Pattern

```typescript
export const openaiFailedResponseHandler = {
  handle: async (response: Response) => {
    const error = await parseJSONBody(response);
    throw new OpenAIError({
      message: error.error?.message ?? 'Unknown error',
      code: error.error?.code,
    });
  },
};
```

---

## 6. Image Generation

**Priority:** CORE | **Files:** ~6 core files | **Providers:** Multiple

### Core API

```typescript
export async function generateImage({
  model: modelArg,
  prompt: promptArg,
  n = 1,
  maxImagesPerCall,
  size,
  aspectRatio,
  seed,
  providerOptions,
  maxRetries: maxRetriesArg,
  abortSignal,
  headers,
}: {
  model: ImageModel;
  prompt: GenerateImagePrompt;
}): Promise<GenerateImageResult>
```

### Automatic Parallelization

Requests are automatically batched when `n > maxImagesPerCall`:

```typescript
const callCount = Math.ceil(n / maxImagesPerCallWithDefault);
const results = await Promise.all(
  callImageCounts.map(async callImageCount =>
    retry(() => model.doGenerate({ prompt, n: callImageCount, ... }))
  )
);
```

### Prompt Normalization

```typescript
function normalizePrompt(prompt: GenerateImagePrompt): Pick<ImageModelV4CallOptions, 'prompt' | 'files' | 'mask'> {
  if (typeof prompt === 'string') {
    return { prompt, files: undefined, mask: undefined };
  }
  return {
    prompt: prompt.text,
    files: prompt.images.map(toImageModelV4File),
    mask: prompt.mask ? toImageModelV4File(prompt.mask) : undefined,
  };
}
```

### Provider Metadata Pattern

Allows providers to return provider-specific data without core SDK changes:

```typescript
providerMetadata: {
  openai: {
    images: [{
      revisedPrompt?: string;  // Provider-specific
    }]
  }
}
```

---

## 7. Embeddings

**Priority:** CORE | **Files:** 6 core files | **Providers:** 30+

### API Surface

**`embed(model, value)`** -- Single embedding
**`embedMany(model, values, options?)`** -- Batch with automatic chunking

### Automatic Chunking

`embedMany` intelligently splits large requests:

```typescript
interface EmbeddingModelV4 {
  maxEmbeddingsPerCall: number | undefined | PromiseLike<number | undefined>;
  supportsParallelCalls: boolean | PromiseLike<boolean>;
}
```

### Polish Level: HIGHEST

Embeddings are the most mature secondary feature:
- Full telemetry integration
- Event callbacks (onStart, onFinish)
- Automatic request chunking with parallel execution
- Comprehensive test coverage (1736 lines across embed.test.ts and embed-many.test.ts)
- Usage tracking
- Warning collection

### Provider Support

30+ providers including OpenAI, Google Vertex, Azure OpenAI, Cohere, Amazon Bedrock, Fireworks, Mistral, TogetherAI.

---

## SECONDARY FEATURES

## 8. Speech/Audio Generation

**Priority:** SECONDARY | **Providers:** ~6 | **Export:** `experimental_generateSpeech`

### Function Signature

```typescript
async function generateSpeech({
  model: SpeechModel,
  text: string,
  voice?: string,
  outputFormat?: 'mp3' | 'wav' | (string & {}),
  instructions?: string,  // "Speak in a slow and steady tone"
  speed?: number,
  language?: string,  // ISO 639-1 or "auto"
  providerOptions,
  maxRetries,
  abortSignal,
  headers,
}): Promise<SpeechResult>
```

### Maturity Gap

| Feature | Text/Embeddings | Speech |
|---------|-----------------|--------|
| Telemetry | Full | **NOT SUPPORTED** |
| onStart callback | Yes | **NOT SUPPORTED** |
| onFinish callback | Yes | **NOT SUPPORTED** |
| Usage tracking | Yes | No |
| Test coverage | ~1700 lines | 300 lines |
| Export prefix | None | `experimental_` |

### Provider Support

~6 providers: OpenAI, ElevenLabs, LMNT, Hume, Deepgram, Fal.

### Audio File Handling

- Media type detection via magic bytes (`detectMediaType` with `audioMediaTypeSignatures`)
- `DefaultGeneratedAudioFile` extends `GeneratedFile` with format property

---

## 9. Video Generation

**Priority:** SECONDARY | **Providers:** ~9 | **Export:** `experimental_generateVideo`

### Function Signature

```typescript
async function experimental_generateVideo({
  model: VideoModel,
  prompt: GenerateVideoPrompt,  // string | { image?; text? }
  n?: number,
  maxVideosPerCall?: number,
  aspectRatio?: `${number}:${number}`,
  resolution?: `${number}x${number}`,
  duration?: number,
  fps?: number,
  seed?: number,
  download?: (options) => Promise<{ data: Uint8Array; mediaType: string | undefined }>,
}): Promise<GenerateVideoResult>
```

### Notable Architecture Detail

**Video is NOT in ProviderRegistry.** Unlike `embeddingModel`, `imageModel`, `speechModel`, the `videoModel` is not part of `ProviderRegistryProvider` interface. Video models are only accessible via direct provider methods (e.g., `klingai.videoModel('model-id')`).

This suggests video generation is at an earlier integration stage or considered more provider-specific.

### Maturity Level: MEDIUM-LOW

| Feature | Status |
|---------|--------|
| Telemetry | NOT SUPPORTED |
| onStart callback | NOT SUPPORTED |
| onFinish callback | NOT SUPPORTED |
| Test coverage | 982 lines |
| Export prefix | `experimental_` |

**Technical debt indicator:** `NoVideoGeneratedError` has deprecated `isNoVideoGeneratedError` method and `toJSON` -- suggests older error pattern.

### Features

Despite lower polish, video has interesting capabilities:
- Image-to-video via `prompt.image`
- Automatic URL download (2 GiB limit)
- Multiple video data formats (url, base64, binary)

---

## 10. Speech-to-Text / Transcription

**Priority:** SECONDARY | **Providers:** 9 | **Export:** `experimental_transcribe`

### API Design

```typescript
const result = await transcribe({
  model: deepgram.transcriptionModel('nova-2'),
  audio: audioData,  // DataContent or URL
  providerOptions: { deepgram: { language: 'en' } },
  maxRetries: 2,
  abortSignal,
});
```

### Result Structure

```typescript
interface TranscriptionResult {
  readonly text: string;
  readonly segments: Array<{
    text: string;
    startSecond: number;
    endSecond: number;
  }>;
  readonly language: string | undefined;
  readonly durationInSeconds: number | undefined;
  readonly warnings: Array<Warning>;
  readonly responses: Array<TranscriptionModelResponseMetadata>;
  readonly providerMetadata: Record<string, JSONObject>;
}
```

### Transcription-Specific Patterns

1. **No streaming** -- Pure request-response (audio in, text out)
2. **Media type detection** -- Automatic via magic bytes
3. **Download abstraction** -- `createDownload()` for URL audio (2 GiB default limit)
4. **Segment timing** -- Word-level timing preserved

### Provider Support

9 providers: Deepgram (most features), OpenAI (Whisper), Rev AI, Groq, Gladia, ElevenLabs, AssemblyAI, TogetherAI, KlingAI.

---

## 11. Reranking

**Priority:** SECONDARY | **Providers:** 2 | **Unique Capability:** Generic documents

### Generic Document Support

Rerank handles both text and JSON object documents:

```typescript
export async function rerank<VALUE extends JSONObject | string>({
  model,
  documents,
  query,
  topN,
  // ...
}): Promise<RerankResult<VALUE>>

// Internal conversion
const documentsToSend =
  typeof documents[0] === 'string'
    ? { type: 'text', values: documents as string[] }
    : { type: 'object', values: documents as JSONObject[] };
```

### Result Structure

```typescript
export interface RerankResult<VALUE> {
  readonly originalDocuments: Array<VALUE>;
  readonly rerankedDocuments: Array<VALUE>;  // Sorted by score desc
  readonly ranking: Array<{
    originalIndex: number;
    score: number;
    document: VALUE;
  }>;
}
```

### Notable: Full Telemetry Support

Unlike speech and video, rerank has **first-class telemetry** like core features:
- Uses `getGlobalTelemetryIntegration()`
- Spans: `ai.rerank` (root), `ai.rerank.doRerank` (inner)
- Event system with onStart/onFinish

### Provider Support

Only 2 providers: Cohere, TogetherAI.

---

## 12. Telemetry & Observability

**Priority:** SECONDARY | **Files:** 12 core files | **Implementation:** OpenTelemetry

### Architecture

```
TelemetrySettings (per-call config)
         │
         ▼
getGlobalTelemetryIntegration() --> TelemetryIntegrationRegistry (global)
         │                                    │
         ▼                                    ▼
   CompositeIntegration              registerTelemetryIntegration()
         │
         ▼
OpenTelemetryIntegration + Custom Integrations
```

### TelemetryIntegration Interface

```typescript
export interface TelemetryIntegration {
  onStart?: Listener<OnStartEvent | EmbedOnStartEvent | RerankOnStartEvent>;
  onStepStart?: Listener<OnStepStartEvent>;
  onToolCallStart?: Listener<OnToolCallStartEvent>;
  onToolCallFinish?: Listener<OnToolCallFinishEvent>;
  onChunk?: Listener<OnChunkEvent>;
  onStepFinish?: Listener<OnStepFinishEvent>;
  onEmbedStart?: Listener<EmbedStartEvent>;
  onEmbedFinish?: Listener<EmbedFinishEvent>;
  onRerankStart?: Listener<RerankStartEvent>;
  onRerankFinish?: Listener<RerankFinishEvent>;
  onFinish?: Listener<OnFinishEvent | EmbedOnFinishEvent | RerankOnFinishEvent>;
  onError?: Listener<unknown>;
  executeTool?: <T>(params: { callId, toolCallId, execute }) => PromiseLike<T>;
}
```

### Traced Operations

| Operation | Span Name |
|-----------|-----------|
| generateText | `ai.generateText` |
| streamText | `ai.streamText` |
| embed | `ai.embed` / `ai.embedMany` |
| rerank | `ai.rerank` |

### Dual Convention Support

Attributes follow both AI SDK and GenAI (OpenTelemetry) conventions:

**AI SDK:** `ai.model.provider`, `ai.prompt`, `ai.usage.inputTokens`
**GenAI:** `gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`

---

## Cross-Cutting Analysis

### Error Handling Pattern

All errors extend `AISDKError` with marker pattern:

```typescript
export class NoImageGeneratedError extends AISDKError {
  private readonly [symbol] = true;

  static isInstance(error: unknown): error is NoImageGeneratedError {
    return AISDKError.hasMarker(error, marker);
  }
}
```

### Shared Infrastructure

| Utility | Used By |
|---------|---------|
| `prepareRetries` | All features |
| `detectMediaType` | Speech, Video, Transcription |
| `createDownload` | Video, Transcription, Image |
| `AISDKError` + marker | All error types |
| Zod validation | Object generation, Tools, Provider options |

### Maturity Comparison

| Feature Tier | Features | Telemetry | Callbacks | Providers | Test Coverage |
|-------------|----------|-----------|-----------|-----------|---------------|
| **Tier 1** | Text, Objects, Embeddings | Full | Full | 30-40+ | ~1700+ lines |
| **Tier 2** | Image, UI Hooks, Agents | Partial | Partial | 10-20 | ~500-1000 lines |
| **Tier 3** | Speech, Video, Transcription | None | None | 2-9 | ~300-500 lines |

### Technical Debt Summary

1. **Vue reactivity quirk** -- Requires explicit cloning in `replaceMessage`
2. **Deprecated methods** -- `addToolResult` vs `addToolOutput`, `NoVideoGeneratedError.toJSON`
3. **TODO comments** -- "JSONStringable" in AbstractChat, "AI SDK 6" invalid inputs
4. **Video not in registry** -- Video models only accessible via direct provider methods
5. **Zod 3/4 compatibility** -- Conditional imports throughout
6. **Experimental prefixes** -- Speech, video, transcription still experimental

### Clever Solutions

1. **SerialJobExecutor** -- Prevents race conditions in parallel tool execution
2. **callbacksRef pattern** -- Solves React stale closure problem
3. **Lazy attribute evaluation** -- Telemetry only computes attributes when recording enabled
4. **Context propagation** -- OTel context maintained across nested spans
5. **DelayedPromise** -- Lazy promise resolution for streaming

---

## Key Files Reference

| Feature | Primary Files |
|---------|--------------|
| Text Generation | `generate-text.ts` (1420 lines), `stream-text.ts` (2668 lines) |
| Object Generation | `generate-object.ts` (515 lines), `stream-object.ts` (985 lines) |
| Agent | `tool-loop-agent.ts` (206 lines), `wrap-language-model.ts` (109 lines) |
| UI Hooks | `chat.ts` (Abstract), `use-chat.ts` (React), framework-specific |
| Provider | `language-model-v4.ts`, `provider-utils/` |
| Image | `generate-image.ts`, `image-model-v4.ts` |
| Embeddings | `embed.ts`, `embed-many.ts` |
| Speech | `generate-speech.ts` |
| Video | `generate-video.ts` |
| Transcription | `transcribe.ts` |
| Reranking | `rerank.ts`, `rerank-events.ts` |
| Telemetry | `open-telemetry-integration.ts` (875 lines) |
