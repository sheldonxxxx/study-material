# Feature Analysis: Text Generation, Structured Object Generation, Agent Framework

## Overview

This document provides a deep-dive analysis of three core features in the Vercel AI SDK:
1. Text Generation (`generateText`, `streamText`)
2. Structured Object Generation (`generateObject`, `streamObject` with Zod)
3. Agent Framework (`ToolLoopAgent`, tools, middleware)

---

## Feature 1: Text Generation

### Entry Points

| Function | Location | Purpose |
|----------|----------|---------|
| `generateText` | `packages/ai/src/generate-text/generate-text.ts` | Non-streaming text completion |
| `streamText` | `packages/ai/src/generate-text/stream-text.ts` | Streaming text completion |

### Call Flow: `generateText` to Provider

```
User Code
    │
    ▼
generateText() ──────────────────────────────────────────────────────┐
    │                                                             │
    ▼                                                             │
1. resolveLanguageModel(modelArg) ──────────────────────────────► Model
    │                                                             │
    ▼                                                             │
2. standardizePrompt({ system, prompt, messages })               │
    │ Converts user input to StandardizedPrompt                   │
    │ { system?: string, messages: ModelMessage[] }              │
    ▼                                                             │
3. prepareCallSettings(settings)                                 │
    │ Validates and normalizes settings (temperature, etc.)     │
    ▼                                                             │
4. retry() wrapper for doGenerate()                             │
    │ Handles retries with AbortSignal support                   │
    ▼                                                             │
5. stepModel.doGenerate({                                        │
        prompt: LanguageModelV4Prompt,    ◄───────────────────────┤
        tools: stepTools,                                        │
        toolChoice: stepToolChoice,                             │
        responseFormat: output?.responseFormat,                 │
        ...callSettings                                          │
      })                                                         │
    │                                                             │
    └─────────────────────────────────────────────────────────────┘
         │
         ▼
    LanguageModelV4 (Provider Implementation)
         │
         ├── doGenerate() -> Promise<LanguageModelV4GenerateResult>
         │
         └── doStream() -> Promise<LanguageModelV4StreamResult>
```

### Key Architecture Patterns

#### Step Loop Pattern

`generateText` implements a **step loop** that continues until stop conditions are met:

```typescript
// From generate-text.ts (lines 668-1024)
do {
  // 1. Prepare step settings via optional prepareStep() callback
  const prepareStepResult = await prepareStep?.({ model, steps, stepNumber, messages });

  // 2. Convert prompt to provider format
  const promptMessages = await convertToLanguageModelPrompt({ prompt, supportedUrls, download });

  // 3. Call the language model
  currentModelResponse = await retry(async () => {
    return stepModel.doGenerate({
      ...callSettings,
      tools: stepTools,
      toolChoice: stepToolChoice,
      responseFormat: await output?.responseFormat,
      prompt: promptMessages,
      ...
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
    clientToolOutputs.push(...await executeTools({ toolCalls: clientToolCalls, ... }));
  }

  // 6. Add tool results to messages for next iteration
  responseMessages.push(...await toResponseMessages({ content: stepContent, tools }));

  // 7. Check stop condition
} while (hasClientToolCalls && !isStopConditionMet({ stopConditions, steps }));
```

#### Stop Conditions

The `stopWhen` parameter accepts conditions that determine when to stop the step loop:

```typescript
// From stop-condition.ts
export const stepCountIs = (count: number): StopCondition => ({
  check({ steps }) {
    return steps.length >= count;
  }
});
```

Default stop condition: `stepCountIs(1)` (single step, no tools).

#### Tool Call Handling

Tools are defined using the `tool()` helper from `@ai-sdk/provider-utils`:

```typescript
const myTool = tool({
  name: 'weather',
  description: 'Get weather for a location',
  parameters: z.object({ location: z.string() }),
  execute: async ({ location }) => {
    return `The weather in ${location} is sunny.`;
  }
});
```

#### Output Abstraction

Text output uses the `Output` abstraction for parsing structured results:

```typescript
// From output.ts
export const text = (): Output<string, string, never> => ({
  name: 'text',
  responseFormat: undefined, // Regular text, no structured output
  async parseCompleteOutput({ text }, context) {
    return text;
  }
});
```

### Error Handling

1. **Retry Logic**: Uses `prepareRetries()` with configurable `maxRetries` and `abortSignal`
2. **Gateway Error Wrapping**: All errors go through `wrapGatewayError()` for consistent error format
3. **Abort Handling**: Multiple AbortSignals merged via `mergeAbortSignals()`
4. **Timeout Management**: Per-step and per-chunk timeouts supported

### Memory Management

The `experimental_include` option controls what data is retained in step results:

```typescript
experimental_include?: {
  requestBody?: boolean;  // Can be large (images)
  responseBody?: boolean;
}
```

### Testing Pattern

Tests use `MockLanguageModelV4` for controlled testing:

```typescript
const model = new MockLanguageModelV4({
  doGenerate: {
    ...dummyResponseValues,
    content: [{ type: 'text', text: 'Hello, world!' }],
  },
});

const result = await generateText({ model, prompt: 'prompt' });
expect(result.text).toBe('Hello, world!');
```

---

## Feature 2: Structured Object Generation

### Entry Points

| Function | Location | Purpose |
|----------|----------|---------|
| `generateObject` | `packages/ai/src/generate-object/generate-object.ts` | Non-streaming object generation |
| `streamObject` | `packages/ai/src/generate-object/stream-object.ts` | Streaming object generation |

### Call Flow: `generateObject` to Provider

```
User Code
    │
    ▼
generateObject({ model, schema: z.object({...}), prompt })
    │
    ▼
1. resolveLanguageModel(modelArg)
    │
    ▼
2. validateObjectGenerationInput() - validates schema, output type
    │
    ▼
3. getOutputStrategy() - creates appropriate strategy based on output type
    │ Returns: ObjectOutputStrategy | ArrayOutputStrategy | EnumOutputStrategy | NoSchemaOutputStrategy
    ▼
4. model.doGenerate({
      responseFormat: {
        type: 'json',
        schema: await outputStrategy.jsonSchema(),  // Zod schema converted to JSON Schema
        name: schemaName,
        description: schemaDescription,
      },
      prompt: promptMessages,
      ...
    })
    │
    ▼
5. extractTextContent(result.content) - get raw JSON string
    │
    ▼
6. parseAndValidateObjectResultWithRepair(result, outputStrategy, repairText, context)
    │ Attempts repair if parsing fails
    ▼
7. Returns GenerateObjectResult
```

### Output Strategy Pattern

The `OutputStrategy` interface handles different output modes:

```typescript
// From output-strategy.ts
interface OutputStrategy<PARTIAL, RESULT, ELEMENT_STREAM> {
  readonly type: 'object' | 'array' | 'enum' | 'no-schema';

  jsonSchema(): Promise<JSONSchema7 | undefined>;

  validatePartialResult({ value, textDelta, ... }): Promise<ValidationResult<{ partial: PARTIAL; textDelta: string }>>;
  validateFinalResult(value, context): Promise<ValidationResult<RESULT>>;

  createElementStream(originalStream): ELEMENT_STREAM;
}
```

### Object Output Strategy

```typescript
const objectOutputStrategy = <OBJECT>(schema: Schema<OBJECT>) => ({
  type: 'object',
  jsonSchema: async () => await schema.jsonSchema,

  validatePartialResult({ value, textDelta }) {
    // No validation for partial results - just return as-is
    return { success: true, value: { partial: value as DeepPartial<OBJECT>, textDelta } };
  },

  validateFinalResult(value) {
    return safeValidateTypes({ value, schema });
  },
  ...
});
```

### Array Output Strategy

Arrays are wrapped in an object structure since most LLMs don't generate arrays directly:

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

`streamObject` uses `parsePartialJson` to incrementally parse and validate:

```typescript
// From stream-object.ts
const { value: currentObjectJson, state: parseState } = await parsePartialJson(accumulatedText);

if (currentObjectJson !== undefined && !isDeepEqualData(latestObjectJson, currentObjectJson)) {
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
```

### Repair Text Pattern

When JSON parsing fails, an optional `repairText` function can fix the output:

```typescript
// From generate-object.ts
const object = await parseAndValidateObjectResultWithRepair(
  result,
  outputStrategy,
  repairText,  // Optional repair function
  { response, usage, finishReason }
);
```

### Zod Integration

The SDK uses `@ai-sdk/provider-utils` for Zod schema handling:

```typescript
import { asSchema, safeValidateTypes } from '@ai-sdk/provider-utils';
import { z } from 'zod/v4';

// Zod schema -> JSON Schema for provider
jsonSchema: async () => await schema.jsonSchema

// Safe validation with error handling
validateFinalResult(value) {
  return safeValidateTypes({ value, schema });
}
```

### Enum Handling

Enums are wrapped similarly to arrays:

```typescript
jsonSchema: async () => ({
  type: 'object',
  properties: {
    result: { type: 'string', enum: enumValues }
  },
  required: ['result']
})
```

---

## Feature 3: Agent Framework

### Entry Points

| Component | Location | Purpose |
|-----------|----------|---------|
| `ToolLoopAgent` | `packages/ai/src/agent/tool-loop-agent.ts` | Main agent implementation |
| `Agent` interface | `packages/ai/src/agent/agent.ts` | Agent contract |
| `wrapLanguageModel` | `packages/ai/src/middleware/wrap-language-model.ts` | Middleware wrapper |

### ToolLoopAgent Architecture

`ToolLoopAgent` is a **thin wrapper** around `generateText`/`streamText`:

```typescript
// From tool-loop-agent.ts
class ToolLoopAgent implements Agent {
  async generate({ ...options }) {
    return generateText({
      ...(await this.prepareCall(options)),
      stopWhen: this.settings.stopWhen ?? stepCountIs(20), // Default 20 steps
      ...callbacks
    });
  }

  async stream({ ...options }) {
    return streamText({
      ...(await this.prepareCall(options)),
      stopWhen: this.settings.stopWhen ?? stepCountIs(20),
      ...callbacks
    });
  }
}
```

### Agent Interface

```typescript
// From agent.ts
export interface Agent<CALL_OPTIONS, TOOLS extends ToolSet, OUTPUT extends Output> {
  readonly version: 'agent-v1';
  readonly id: string | undefined;
  readonly tools: TOOLS;

  generate(options: AgentCallParameters<CALL_OPTIONS, TOOLS>): Promise<GenerateTextResult<TOOLS, OUTPUT>>;
  stream(options: AgentStreamParameters<CALL_OPTIONS, TOOLS>): Promise<StreamTextResult<TOOLS, OUTPUT>>;
}
```

### Tool Execution Loop

The tool loop is implemented by `generateText`/`streamText`:

1. LLM generates response with potential tool calls
2. If tool calls present: execute tools, add results to messages, loop back to step 1
3. If no tool calls or stop condition met: return response

### Middleware Framework

#### LanguageModelMiddleware Interface

```typescript
// From language-model-middleware.ts (provider package)
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

#### Middleware Application

```typescript
// From wrap-language-model.ts
export const wrapLanguageModel = ({ model, middleware, modelId, providerId }) => {
  // Middleware is applied in reverse order (first middleware is outermost)
  return [...asArray(middlewareArg)]
    .reverse()
    .reduce((wrappedModel, middleware) => {
      return doWrap({ model: wrappedModel, middleware, modelId, providerId });
    }, model);
};
```

#### Built-in Middleware Examples

**extractJsonMiddleware**: Extracts JSON from markdown code blocks:

```typescript
transformParams: ({ params }) => ({
  ...params,
  prompt: params.prompt.map(msg => /* extract JSON from text parts */)
})
```

**simulateStreamingMiddleware**: Makes non-streaming calls appear as streams:

```typescript
wrapGenerate: ({ doGenerate }) => {
  const result = await doGenerate();
  // Convert to streaming format
}
```

**addToolInputExamplesMiddleware**: Adds examples to tool prompts for better parsing:

```typescript
transformParams: ({ params }) => ({
  ...params,
  tools: params.tools?.map(tool => /* add examples */)
})
```

### Tool Types

#### Tool Definition

```typescript
// From @ai-sdk/provider-utils
const myTool = tool({
  name: 'calculator',
  description: 'A calculator tool',
  parameters: z.object({ a: z.number(), b: z.number() }),
  execute: async ({ a, b }) => a + b,
  // Optional streaming support:
  stream: async function* ({ a, b }) {
    yield { type: 'preliminary', output: `Computing ${a} + ${b}...` };
    yield { type: 'output', output: a + b };
  }
});
```

#### Tool Lifecycle

```typescript
// From execute-tool-call.ts
export async function executeToolCall({ toolCall, tools, ... }) {
  const tool = tools?.[toolName];

  // 1. Notify start
  await notify({ event: { ...baseCallbackEvent }, callbacks: onToolCallStart });

  // 2. Execute with timeout
  const toolAbortSignal = AbortSignal.timeout(toolTimeoutMs);
  output = await executeTool({
    execute: tool.execute.bind(tool),
    input,
    options: { toolCallId, messages, abortSignal: toolAbortSignal, experimental_context }
  });

  // 3. Handle streaming results
  for await (const part of stream) {
    if (part.type === 'preliminary') {
      onPreliminaryToolResult?.({ type: 'tool-result', output: part.output, preliminary: true });
    } else {
      output = part.output;
    }
  }

  // 4. Notify finish
  await notify({ event: { ...baseCallbackEvent, success: true, output, durationMs }, callbacks: onToolCallFinish });

  return { type: 'tool-result', toolCallId, toolName, input, output };
}
```

---

## Cross-Cutting Concerns

### Telemetry Integration

All features integrate with OpenTelemetry via a consistent pattern:

```typescript
// From generate-text.ts
const tracer = getTracer(telemetry);
const globalTelemetry = createGlobalTelemetry({ integrations: telemetry?.integrations });

// Spans created for: ai.generateText, ai.generateText.doGenerate
// Attributes: operationId, model, prompt, usage, finish reason, etc.
```

### Provider Abstraction

All provider communication goes through `LanguageModelV4` interface:

```typescript
// From packages/provider/src/language-model/v4/language-model-v4.ts
type LanguageModelV4 = {
  readonly specificationVersion: 'v4';
  readonly provider: string;
  readonly modelId: string;
  supportedUrls: PromiseLike<Record<string, RegExp[]>> | Record<string, RegExp[]>;

  doGenerate(options: LanguageModelV4CallOptions): PromiseLike<LanguageModelV4GenerateResult>;
  doStream(options: LanguageModelV4CallOptions): PromiseLike<LanguageModelV4StreamResult>;
};
```

### Error Handling Patterns

1. **AISDKError base class** with `isInstance()` pattern for error identification
2. **Specific error types**: `NoObjectGeneratedError`, `MissingToolResultsError`, `InvalidPromptError`
3. **Error wrapping**: `wrapGatewayError()` for provider errors
4. **Abort handling**: `isAbortError()` utility

### Prompt Standardization

The SDK normalizes prompts before sending to providers:

```typescript
// From standardize-prompt.ts
export async function standardizePrompt(prompt: Prompt): Promise<StandardizedPrompt> {
  // Validates: prompt XOR messages is set
  // Validates: system is string or SystemModelMessage[]
  // Validates: messages match ModelMessage[] schema
  // Returns: { system, messages }
}
```

### Stream Processing

Streaming uses Web Streams API with TransformStreams for processing:

```typescript
// Pattern used throughout
const transformedStream = originalStream
  .pipeThrough(new TransformStream({ transform(chunk, controller) { ... } }))
  .pipeThrough(anotherTransform);
```

### Retry Logic

```typescript
// From prepare-retries.ts
const { maxRetries, retry } = prepareRetries({ maxRetries, abortSignal });

// retry() wraps operations and handles exponential backoff
const result = await retry(() => someOperation());
```

---

## Notable Implementation Details

### Clever Solutions

1. **DelayedPromise**: Used for lazy promise resolution in streaming results
   ```typescript
   // From @ai-sdk/provider-utils
   class DelayedPromise<T> {
     promise: Promise<T>;
     resolve: (value: T) => void;
     reject: (error: unknown) => void;
   }
   ```

2. **StitchableStream**: Allows multiple streams to be combined
   ```typescript
   // From create-stitchable-stream.ts
   const stitchableStream = createStitchableStream<TextStreamPart>();
   stitchableStream.addStream(stream1);
   stitchableStream.addStream(stream2);
   stitchableStream.close();
   ```

3. **Tool Call Parsing with Repair**: Invalid tool calls can be repaired
   ```typescript
   // From parse-tool-call.ts
   const toolCall = await parseToolCall({ toolCall, tools, repairToolCall, ... });
   ```

### Technical Debt / Known Issues

1. **TODO in code** (generate-text.ts line 842):
   ```typescript
   // TODO AI SDK 6: invalid inputs should not require output parts
   ```

2. **Deferred Tool Results**: Provider-executed tools that support deferred results need special handling for multi-turn scenarios

3. **Complex Merge Pattern**: Tool approval handling involves complex message manipulation

### Performance Considerations

1. **Download Once**: URL downloads happen once per prompt, not per model call
2. **Parallel Tool Execution**: Multiple tools can be executed in parallel
3. **Memory Control**: `experimental_include` prevents memory bloat from large payloads

---

## Testing Patterns

### MockLanguageModelV4

Tests use a mock that implements `LanguageModelV4`:

```typescript
const model = new MockLanguageModelV4({
  doGenerate: { ... },
  doStream: { ... }
});
```

### Fake Timers

```typescript
beforeEach(() => {
  vi.useFakeTimers();
  vi.setSystemTime(new Date(0));
});

afterEach(() => {
  vi.useRealTimers();
});
```

### Spy on Modules

```typescript
const logWarningsSpy = vitest.spyOn(logWarningsModule, 'logWarnings').mockImplementation(() => {});
```

---

## File Summary

| File | Lines | Purpose |
|------|-------|---------|
| generate-text.ts | 1420 | Main generateText implementation with step loop |
| stream-text.ts | 2668 | StreamText with complex streaming transformations |
| stream-model-call.ts | 419 | Bridge between model streaming and SDK stream format |
| generate-object.ts | 515 | Object generation with Zod validation |
| stream-object.ts | 985 | Streaming object generation |
| output-strategy.ts | 416 | Strategy pattern for different output modes |
| tool-loop-agent.ts | 206 | Agent wrapper around generateText/streamText |
| agent.ts | 157 | Agent interface definition |
| wrap-language-model.ts | 109 | Middleware composition |
| execute-tool-call.ts | 191 | Tool execution with callbacks |
| standardize-prompt.ts | 100 | Prompt normalization |
| convert-to-language-model-prompt.ts | 549 | Message format conversion |
