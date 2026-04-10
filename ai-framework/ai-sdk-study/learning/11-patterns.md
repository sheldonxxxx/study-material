# Design Patterns in Vercel AI SDK

## Pattern Index

1. [Layered Provider Abstraction](#1-layered-provider-abstraction)
2. [Adapter Pattern](#2-adapter-pattern)
3. [OutputStrategy Pattern](#3-outputstrategy-pattern)
4. [Step Loop Pattern](#4-step-loop-pattern)
5. [Middleware System](#5-middleware-system)
6. [Delayed Promise Pattern](#6-delayed-promise-pattern)
7. [Framework Abstraction](#7-framework-abstraction)
8. [Interface Versioning](#8-interface-versioning)
9. [Error Marker Pattern](#9-error-marker-pattern)
10. [Serial Job Executor](#10-serial-job-executor)
11. [Prompt Standardization](#11-prompt-standardization)
12. [Streaming with TransformStreams](#12-streaming-with-transformstreams)
13. [Tool Definition Builder](#13-tool-definition-builder)
14. [Stitchable Stream](#14-stitchable-stream)
15. [Callbacks Ref Pattern](#15-callbacks-ref-pattern)

---

## 1. Layered Provider Abstraction

**Problem:** Need to support multiple AI providers with a unified API while allowing provider-specific implementations.

**Solution:** Layer architecture separating specifications, utilities, implementations, and user-facing APIs.

**Evidence:**
```
packages/ai (Core SDK)
       |
packages/provider (Interface Specifications)
       |
packages/provider-utils (Shared Utilities)
       |
packages/<provider-*> (33 Provider Implementations)
```

**Files:**
- `packages/provider/src/language-model/v4/language-model-v4.ts` - Core interface
- `packages/provider-utils/src/post-to-api.ts` - HTTP utilities
- `packages/openai/src/openai-provider.ts` - Provider implementation

**Code:**
```typescript
// packages/ai/src/generate-text/generate-text.ts
// Core uses LanguageModelV4 interface
const response = await stepModel.doGenerate({
  prompt: promptMessages,
  tools: stepTools,
  // ...
});
```

---

## 2. Adapter Pattern

**Problem:** Each AI provider has different API formats for requests and responses.

**Solution:** Concrete adapter classes implement `LanguageModelV4` interface, converting to/from provider-specific formats.

**Evidence:**
```typescript
// packages/openai/src/chat/openai-chat-language-model.ts
export class OpenAIChatLanguageModel implements LanguageModelV4 {
  readonly specificationVersion = 'v4';
  readonly modelId: OpenAIChatModelId;

  async doGenerate(options: LanguageModelV4CallOptions) {
    // 1. Convert prompt to OpenAI format
    const { messages } = convertToOpenAIChatMessages({ prompt: options.prompt });

    // 2. Make provider-specific API call
    const response = await postJsonToApi({
      url: this.config.url({ path: '/chat/completions' }),
      headers,
      body: { model: this.modelId, messages, ... }
    });

    // 3. Convert response to standard format
    return {
      content: response.choices.map(choice => ({
        type: 'text',
        text: choice.message.content
      })),
      finishReason: mapOpenAIFinishReason(response.choices[0].finish_reason),
      usage: convertOpenAIChatUsage(response.usage),
      warnings: [],
    };
  }
}
```

**Pattern benefit:** Adding a new provider only requires implementing `LanguageModelV4`.

---

## 3. OutputStrategy Pattern

**Problem:** Need to handle multiple structured output modes (single object, array, enum, no schema) with different validation requirements.

**Solution:** Strategy pattern with `OutputStrategy` interface for each output mode.

**Evidence:**
```typescript
// packages/ai/src/generate-object/output-strategy.ts
interface OutputStrategy<PARTIAL, RESULT, ELEMENT_STREAM> {
  readonly type: 'object' | 'array' | 'enum' | 'no-schema';

  jsonSchema(): Promise<JSONSchema7 | undefined>;

  validatePartialResult({
    value,
    textDelta,
    latestObject,
    isFirstDelta,
    isFinalDelta,
  }): Promise<ValidationResult<{ partial: PARTIAL; textDelta: string }>>;

  validateFinalResult(
    value: unknown,
    context: ObjectValidationContext,
  ): Promise<ValidationResult<RESULT>>;

  createElementStream(originalStream: ReadableStream<string>): ELEMENT_STREAM;
}
```

**Implementations:**
```typescript
// Object strategy
const objectOutputStrategy = <OBJECT>(schema: Schema<OBJECT>) => ({
  type: 'object',
  jsonSchema: async () => await schema.jsonSchema,
  validatePartialResult({ value, textDelta }) {
    return { success: true, value: { partial: value as DeepPartial<OBJECT>, textDelta } };
  },
  validateFinalResult(value) {
    return safeValidateTypes({ value, schema });
  },
});

// Array strategy wraps in { elements: [] }
const arrayOutputStrategy = <OBJECT>(itemSchema: Schema<OBJECT>) => ({
  type: 'array',
  jsonSchema: async () => ({
    type: 'object',
    properties: {
      elements: { type: 'array', items: itemSchema.jsonSchema }
    },
    required: ['elements'],
  }),
  // ...
});
```

**Files:**
- `packages/ai/src/generate-object/output-strategy.ts` (416 lines)
- `packages/ai/src/generate-object/generate-object.ts`
- `packages/ai/src/generate-object/stream-object.ts`

---

## 4. Step Loop Pattern

**Problem:** Tool execution requires multiple model calls (generate -> execute tools -> generate again) until no more tools are called.

**Solution:** `do-while` loop in `generateText` that continues until stop conditions are met.

**Evidence:**
```typescript
// packages/ai/src/generate-text/generate-text.ts (lines 668-1024)
do {
  // 1. Optional callback to prepare step
  const prepareStepResult = await prepareStep?.({
    model,
    steps,
    stepNumber,
    messages
  });

  // 2. Convert prompt to provider format
  const promptMessages = await convertToLanguageModelPrompt({
    prompt,
    supportedUrls,
    download
  });

  // 3. Call the language model
  currentModelResponse = await retry(async () => {
    return stepModel.doGenerate({
      ...callSettings,
      tools: stepTools,
      toolChoice: stepToolChoice,
      responseFormat: await output?.responseFormat,
      prompt: promptMessages,
    });
  });

  // 4. Parse tool calls from response
  const stepToolCalls = await Promise.all(
    currentModelResponse.content
      .filter((part): part is LanguageModelV4ToolCall => part.type === 'tool-call')
      .map(toolCall => parseToolCall({ toolCall, tools, repairToolCall, system, messages }))
  );

  // 5. Execute tool calls
  if (tools != null) {
    clientToolOutputs.push(...await executeTools({
      toolCalls: clientToolCalls,
      // ...
    }));
  }

  // 6. Add tool results to messages for next iteration
  responseMessages.push(...await toResponseMessages({
    content: stepContent,
    tools
  }));

  // 7. Check stop condition
} while (
  hasClientToolCalls &&
  !isStopConditionMet({ stopConditions, steps })
);
```

**Stop Conditions:**
```typescript
// packages/ai/src/generate-text/stop-condition.ts
export const stepCountIs = (count: number): StopCondition => ({
  check({ steps }) {
    return steps.length >= count;
  }
});
```

**Files:**
- `packages/ai/src/generate-text/generate-text.ts` (1420 lines)
- `packages/ai/src/generate-text/stop-condition.ts`

---

## 5. Middleware System

**Problem:** Need to add cross-cutting concerns (logging, caching, default settings) without modifying core logic.

**Solution:** Middleware wraps language models to transform parameters and wrap responses.

**Evidence:**
```typescript
// packages/ai/src/middleware/wrap-language-model.ts
export function wrapLanguageModel({ model, middleware }) {
  // Middleware applied in reverse order (first = outermost)
  return [...asArray(middlewareArg)]
    .reverse()
    .reduce((wrappedModel, currentMiddleware) => {
      return doWrap({ model: wrappedModel, middleware: currentMiddleware });
    }, model);
}

function doWrap({ model, middleware }) {
  return {
    specificationVersion: 'v4',
    provider: model.provider,
    modelId: model.modelId,
    supportedUrls: model.supportedUrls,

    async doGenerate(options) {
      // Transform params before calling model
      const transformedOptions =
        middleware.transformParams?.({ model, params: options }) ?? options;

      // Wrap the generate call
      return middleware.wrapGenerate?.({
        model,
        params: transformedOptions,
        fn: () => model.doGenerate(transformedOptions)
      }) ?? model.doGenerate(transformedOptions);
    },

    doStream(options) {
      // Similar for streaming
    }
  };
}
```

**Middleware Interface:**
```typescript
// packages/provider/src/middleware/language-model-middleware.ts
type LanguageModelV4Middleware = {
  readonly specificationVersion: 'v4';

  // Transform parameters before they're passed to the model
  transformParams?: ({ params, type, model }) => LanguageModelV4CallOptions;

  // Wrap the generate operation
  wrapGenerate?: ({ doGenerate, doStream, params, model }) => LanguageModelV4GenerateResult;

  // Wrap the stream operation
  wrapStream?: ({ doGenerate, doStream, params, model }) => LanguageModelV4StreamResult;

  // Override model metadata
  overrideProvider?: ({ model }) => string;
  overrideModelId?: ({ model }) => string;
  overrideSupportedUrls?: ({ model }) => Record<string, RegExp[]>;
};
```

**Built-in Middleware:**
- `add-tool-input-examples-middleware.ts` - Adds examples to tool prompts
- `default-settings-middleware.ts` - Applies default settings
- `extract-json-middleware.ts` - Extracts JSON from markdown
- `extract-reasoning-middleware.ts` - Extracts reasoning content
- `simulate-streaming-middleware.ts` - Makes non-streaming look like streaming

**Files:**
- `packages/ai/src/middleware/wrap-language-model.ts` (109 lines)
- `packages/provider/src/middleware/language-model-middleware.ts`

---

## 6. Delayed Promise Pattern

**Problem:** Need lazy promise resolution for cases where promise creation and resolution happen at different times.

**Solution:** `DelayedPromise` class that exposes resolve/reject methods before the promise settles.

**Evidence:**
```typescript
// packages/provider-utils/src/delayed-promise.ts
export class DelayedPromise<T> {
  promise: Promise<T>;
  resolve: (value: T) => void;
  reject: (error: unknown) => void;

  constructor() {
    this.promise = new Promise<T>((resolve, reject) => {
      this.resolve = resolve;
      this.reject = reject;
    });
  }
}

// Usage: Streaming result with delayed completion
class StreamingResult {
  private chunks: string[] = [];
  private fullfilled = new DelayedPromise<string>();

  addChunk(chunk: string) {
    this.chunks.push(chunk);
  }

  async finish() {
    this.fullfilled.resolve(this.chunks.join(''));
  }

  getText(): Promise<string> {
    return this.fullfilled.promise;
  }
}
```

**Files:**
- `packages/provider-utils/src/delayed-promise.ts`

---

## 7. Framework Abstraction

**Problem:** Need consistent chat UI API across React, Vue, Svelte, and Angular with different reactivity systems.

**Solution:** Abstract base class (`AbstractChat`) with framework-specific state implementations (`ChatState`).

**Evidence:**
```typescript
// packages/ai/src/ui/chat.ts
export abstract class AbstractChat<UI_MESSAGE extends UIMessage> {
  readonly id: string;
  protected state: ChatState<UI_MESSAGE>;
  private transport: ChatTransport<UI_MESSAGE>;
  private jobExecutor = new SerialJobExecutor();

  abstract sendMessage(...): void;
  abstract regenerate(...): void;
  abstract stop(...): void;
}

export interface ChatState<UI_MESSAGE extends UIMessage> {
  status: ChatStatus;
  error: Error | undefined;
  messages: UI_MESSAGE[];
  pushMessage: (message: UI_MESSAGE) => void;
  popMessage: () => void;
  replaceMessage: (index: number, message: UI_MESSAGE) => void;
  snapshot: <T>(thing: T) => T;
}
```

**Framework Implementations:**

```typescript
// React - packages/react/src/chat.react.ts
class ReactChatState<UI_MESSAGE extends UIMessage> implements ChatState<UI_MESSAGE> {
  // Uses useSyncExternalStore
}

// Vue - packages/vue/src/chat.vue.ts
class VueChatState<UI_MESSAGE extends UIMessage> implements ChatState<UI_MESSAGE> {
  // Uses ref and reactive
  replaceMessage = (index: number, message: UI_MESSAGE) => {
    this.messagesRef.value[index] = { ...message }; // Note: cloning required
  };
}

// Svelte - packages/svelte/src/chat.svelte.ts
class SvelteChatState<UI_MESSAGE extends UIMessage> implements ChatState<UI_MESSAGE> {
  messages = $state<UI_MESSAGE[]>([]);
  snapshot = <T>(thing: T): T => $state.snapshot(thing);
}

// Angular - packages/angular/src/lib/chat.ng.ts
class AngularChatState<UI_MESSAGE extends UIMessage> implements ChatState<UI_MESSAGE> {
  readonly #messages = signal<UI_MESSAGE[]>([]);
  snapshot = <T>(thing: T): T => structuredClone(thing);
}
```

**Files:**
- `packages/ai/src/ui/chat.ts` - AbstractChat
- `packages/react/src/chat.react.ts`
- `packages/vue/src/chat.vue.ts`
- `packages/svelte/src/chat.svelte.ts`
- `packages/angular/src/lib/chat.ng.ts`

---

## 8. Interface Versioning

**Problem:** Need to evolve interfaces without breaking existing provider implementations.

**Solution:** Multiple interface versions coexist with adapter functions converting older versions to newer.

**Evidence:**
```typescript
// packages/ai/src/model/as-language-model-v4.ts
export function asLanguageModelV4(
  languageModel: LanguageModel | LanguageModelV3 | LanguageModelV2
): LanguageModelV4 {
  if (isLanguageModelV4(languageModel)) {
    return languageModel;
  }

  if (isLanguageModelV3(languageModel)) {
    return convertLanguageModelV3ToV4(languageModel);
  }

  if (isLanguageModelV2(languageModel)) {
    return convertLanguageModelV2ToV4(languageModel);
  }

  // ...
}
```

**Version Detection:**
```typescript
function isLanguageModelV4(
  value: unknown
): value is LanguageModelV4 {
  return (
    isObject(value) &&
    value.specificationVersion === 'v4' &&
    typeof value.doGenerate === 'function' &&
    typeof value.doStream === 'function'
  );
}
```

**Files:**
- `packages/ai/src/model/as-language-model-v4.ts`
- `packages/ai/src/model/as-embedding-model-v4.ts`
- `packages/ai/src/model/as-provider-v4.ts`

---

## 9. Error Marker Pattern

**Problem:** Need precise error identification for error handling without `instanceof` checks across package boundaries.

**Solution:** Each error type has a unique symbol marker with static `isInstance` method.

**Evidence:**
```typescript
// packages/ai/src/error/ai-sdk-error.ts
export class AISDKError extends Error {
  private readonly [errorMarker] = true;

  static isInstance(error: unknown): error is AISDKError {
    return isObject(error) && error[errorMarker] === true;
  }

  static hasMarker(error: unknown, marker: symbol): boolean {
    return isObject(error) && error[marker] === true;
  }
}

// packages/ai/src/error/no-such-model-error.ts
export class NoSuchModelError extends AISDKError {
  private readonly [noSuchModelErrorMarker] = true;

  static isInstance(error: unknown): error is NoSuchModelError {
    return AISDKError.hasMarker(error, noSuchModelErrorMarker);
  }
}
```

**Files:**
- `packages/ai/src/error/ai-sdk-error.ts`
- `packages/ai/src/error/no-such-model-error.ts`
- `packages/ai/src/error/missing-tool-results-error.ts`
- `packages/ai/src/error/no-object-generated-error.ts`

---

## 10. Serial Job Executor

**Problem:** When multiple tools are called in parallel, results must be processed in order to avoid race conditions with message state.

**Solution:** `SerialJobExecutor` ensures sequential processing even when tools execute in parallel.

**Evidence:**
```typescript
// packages/ai/src/ui/serial-job-executor.ts
class SerialJobExecutor {
  private queue: Job[] = [];

  async execute<T>(job: () => Promise<T>): Promise<T> {
    return new Promise((resolve, reject) => {
      this.queue.push({ job, resolve, reject });
      this.process();
    });
  }

  private async process() {
    if (this.processing || this.queue.length === 0) return;

    this.processing = true;
    const { job, resolve, reject } = this.queue.shift()!;

    try {
      const result = await job();
      resolve(result);
    } catch (error) {
      reject(error);
    } finally {
      this.processing = false;
      this.process(); // Process next
    }
  }
}
```

**Files:**
- `packages/ai/src/ui/serial-job-executor.ts`

---

## 11. Prompt Standardization

**Problem:** Users provide prompts in various formats (string, array of messages, with/without system prompt) that need normalization before provider calls.

**Solution:** Multi-stage prompt conversion pipeline.

**Evidence:**
```typescript
// packages/ai/src/prompt/standardize-prompt.ts
export async function standardizePrompt(prompt: Prompt): Promise<StandardizedPrompt> {
  // Validates: prompt XOR messages is set (not both)
  // Validates: system is string or SystemModelMessage[]
  // Validates: messages match ModelMessage[] schema
  return {
    system: /* normalized system prompt */,
    messages: /* normalized messages array */
  };
}

// packages/ai/src/prompt/convert-to-language-model-prompt.ts
export async function convertToLanguageModelPrompt({
  prompt,
  supportedUrls,
  download
}): Promise<LanguageModelV4Prompt> {
  // Converts StandardizedPrompt to LanguageModelV4Prompt
  // Handles content parts, tool results, etc.
}
```

**Files:**
- `packages/ai/src/prompt/standardize-prompt.ts` (100 lines)
- `packages/ai/src/prompt/convert-to-language-model-prompt.ts` (549 lines)

---

## 12. Streaming with TransformStreams

**Problem:** Need to transform streaming data through multiple processing stages (parse, validate, format).

**Solution:** Web Streams API with `pipeThrough` for composable stream transformation.

**Evidence:**
```typescript
// packages/ai/src/generate-object/stream-object.ts
const { value: currentObjectJson, state: parseState } =
  await parsePartialJson(accumulatedText);

if (currentObjectJson !== undefined) {
  const validationResult = await outputStrategy.validatePartialResult({
    value: currentObjectJson,
    textDelta,
    latestObject,
    isFirstDelta,
    isFinalDelta: parseState === 'successful-parse',
  });

  if (validationResult.success) {
    latestObject = validationResult.value.partial;
    controller.enqueue({ type: 'object', object: latestObject });
  }
}

// packages/ai/src/generate-text/stream-text.ts
const transformedStream = originalStream
  .pipeThrough(new TextDecoderStream())
  .pipeThrough(new TransformStream({
    transform(chunk, controller) {
      // Parse and emit text deltas
    }
  }));
```

**Files:**
- `packages/ai/src/generate-object/stream-object.ts` (985 lines)
- `packages/ai/src/generate-text/stream-text.ts` (2668 lines)

---

## 13. Tool Definition Builder

**Problem:** Tool definitions require schema validation, description, and execution logic with consistent structure.

**Solution:** `tool()` builder function that creates properly typed tool definitions.

**Evidence:**
```typescript
// packages/provider-utils/src/tool.ts
export function tool<T extends Schema>({
  name,
  description,
  parameters,
  execute,
  stream,
}: {
  name: string;
  description: string;
  parameters: T;
  execute: (params: Infer<T>, context: ToolExecutionContext) => Promise<unknown>;
  stream?: (params: Infer<T>, context: ToolExecutionContext) => AsyncGenerator<ToolResultPart>;
}): Tool {
  return {
    name,
    description,
    parameters: parameters.jsonSchema,
    execute: async ({ toolCallId, args, ... }) => {
      const parsed = safeValidateTypes({ value: args, schema: parameters });
      return execute(parsed.value, context);
    },
    stream: stream ? /* wrap stream */ : undefined,
  };
}

// Usage
const weatherTool = tool({
  name: 'weather',
  description: 'Get weather for a location',
  parameters: z.object({ location: z.string() }),
  execute: async ({ location }) => {
    return `The weather in ${location} is sunny.`;
  },
  stream: async function* ({ location }) {
    yield { type: 'preliminary', output: `Looking up ${location}...` };
    yield { type: 'output', output: `The weather in ${location} is sunny.` };
  }
});
```

**Files:**
- `packages/provider-utils/src/tool.ts`
- `packages/ai/src/generate-text/execute-tool-call.ts`

---

## 14. Stitchable Stream

**Problem:** Need to combine multiple streams into one, allowing incremental stream addition before closing.

**Solution:** `createStitchableStream` factory creates a stream that accepts additions.

**Evidence:**
```typescript
// packages/provider-utils/src/create-stitchable-stream.ts
function createStitchableStream<T>(): {
  stream: ReadableStream<T>;
  addStream: (stream: ReadableStream<T>) => void;
  close: () => void;
} {
  const streams: ReadableStream<T>[] = [];
  let controller: ReadableStreamDefaultController<T>;

  return {
    stream: new ReadableStream<T>({
      start(c) { controller = c; },
      pull() {
        // Read from streams sequentially
      }
    }),

    addStream(stream: ReadableStream<T>) {
      streams.push(stream);
    },

    close() {
      // Signal no more streams
    }
  };
}

// Usage
const stitchableStream = createStitchableStream<TextStreamPart>();
stitchableStream.addStream(stream1);
stitchableStream.addStream(stream2);
stitchableStream.close();
```

**Files:**
- `packages/provider-utils/src/create-stitchable-stream.ts`

---

## 15. Callbacks Ref Pattern

**Problem:** React hooks face stale closure issues when callbacks are captured in `useEffect` dependencies but need to reference latest values.

**Solution:** Store callbacks in a ref that updates on every render but maintains stable object identity for the hook.

**Evidence:**
```typescript
// packages/react/src/use-chat.ts
export function useChat<UI_MESSAGE extends UIMessage = UIMessage>({
  experimental_throttle: throttleWaitMs,
  resume = false,
  ...options
}: UseChatOptions<UI_MESSAGE> = {}): UseChatHelpers<UI_MESSAGE> {
  // Callbacks ref to avoid stale closures
  const callbacksRef = useRef(
    !('chat' in options)
      ? { onToolCall, onData, onFinish, onError, onStart }
      : {}
  );

  // Update callbacks on every render without triggering re-subscription
  callbacksRef.current = !('chat' in options)
    ? { onToolCall, onData, onFinish, onError, onStart }
    : {};

  // Create/retrieve Chat instance
  const chatRef = useRef<Chat<UI_MESSAGE>>(
    'chat' in options
      ? options.chat
      : new Chat({ ...optionsWithCallbacks, callbacks: callbacksRef.current })
  );

  // Subscribe to state changes
  const messages = useSyncExternalStore(
    subscribeToMessages,
    () => chatRef.current.messages,
    () => chatRef.current.messages,
  );

  return { ...state, sendMessage: chatRef.current.sendMessage };
}
```

**Pattern benefit:** Chat instance subscribes once, but callbacks always reference latest values.

**Files:**
- `packages/react/src/use-chat.ts`

---

## Pattern Summary Table

| Pattern | Purpose | Key File |
|---------|---------|----------|
| Layered Provider Abstraction | Separation of concerns | `packages/provider/src/` |
| Adapter Pattern | Provider interoperability | `packages/openai/src/chat/` |
| OutputStrategy | Structured output modes | `packages/ai/src/generate-object/output-strategy.ts` |
| Step Loop | Tool execution | `packages/ai/src/generate-text/generate-text.ts` |
| Middleware System | Cross-cutting concerns | `packages/ai/src/middleware/wrap-language-model.ts` |
| Delayed Promise | Lazy resolution | `packages/provider-utils/src/delayed-promise.ts` |
| Framework Abstraction | Multi-framework support | `packages/ai/src/ui/chat.ts` |
| Interface Versioning | Backwards compatibility | `packages/ai/src/model/as-*-v4.ts` |
| Error Marker Pattern | Type-safe errors | `packages/ai/src/error/` |
| Serial Job Executor | Ordered parallel execution | `packages/ai/src/ui/serial-job-executor.ts` |
| Prompt Standardization | Input normalization | `packages/ai/src/prompt/` |
| Streaming TransformStreams | Composable stream processing | `packages/ai/src/generate-object/stream-object.ts` |
| Tool Definition Builder | Tool creation | `packages/provider-utils/src/tool.ts` |
| Stitchable Stream | Stream concatenation | `packages/provider-utils/src/create-stitchable-stream.ts` |
| Callbacks Ref | Stale closure solution | `packages/react/src/use-chat.ts` |
