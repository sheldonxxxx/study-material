# My Action Items: Building an AI SDK / Provider Abstraction Layer

## Context

Based on lessons from the Vercel AI SDK research, these are prioritized action items for building a similar provider abstraction layer. Assumes starting from scratch or early-stage development.

**Priority Key:**
- P0: Critical foundation (do first)
- P1: Core functionality (do early)
- P2: Enhancement (do when mature)
- P3: Nice-to-have (consider later)

---

## P0: Foundation

### 1. Define Core Interface Structure

**Rationale:** Vercel AI SDK's success stems from clean interfaces. Starting with interfaces prevents adapter sprawl.

**Action:**
```
packages/
  core/           # User-facing APIs (generateText, etc.)
  interfaces/     # Provider contracts (LanguageModelV4, etc.)
  utils/          # Shared utilities
  providers/      # Provider implementations
```

**Files to create:**
- `packages/interfaces/src/language-model/v1/language-model-v1.ts`
- `packages/interfaces/src/index.ts`

**Pattern to follow:**
```typescript
export type LanguageModelV1 = {
  readonly specificationVersion: 'v1';
  readonly provider: string;
  readonly modelId: string;

  doGenerate(options: LanguageModelV1CallOptions): PromiseLike<LanguageModelV1GenerateResult>;
  doStream(options: LanguageModelV1CallOptions): PromiseLike<LanguageModelV1StreamResult>;
};
```

**Verification:** Interface can be implemented by a mock provider without imports from core.

---

### 2. Implement Secure JSON Parsing

**Rationale:** LLM outputs are untrusted input. Vercel AI SDK explicitly bans `JSON.parse`. Prototype pollution is a real attack vector.

**Action:** Create `packages/utils/src/secure-json-parse.ts`:

```typescript
// Implement or wrap 'secure-json-parse' from fastify
// Include prototype pollution checks
// Export: safeJsonParse, safeStringify
```

**Verification:** `safeJsonParse('{"__proto__":{"admin":true}}')` returns `{}` without pollution.

---

### 3. Create Error Hierarchy

**Rationale:** Vercel AI SDK has 21+ error types with marker pattern. Enables safe error narrowing.

**Action:** Create `packages/interfaces/src/errors/`:

```typescript
// packages/interfaces/src/errors/base-error.ts
export abstract class SDKError extends Error {
  abstract readonly code: string;
  protected abstract readonly marker: symbol;

  static hasMarker(error: unknown, marker: symbol): boolean {
    if (error instanceof Error && marker in error) {
      return true;
    }
    return false;
  }
}

// packages/interfaces/src/errors/api-error.ts
export class APICallError extends SDKError {
  readonly code = 'API_CALL_ERROR';
  readonly isRetryable: boolean;

  static isInstance(error: unknown): error is APICallError {
    return SDKError.hasMarker(error, Symbol.for('vercel.ai.error.APICallError'));
  }
}

// packages/interfaces/src/errors/validation-error.ts
// packages/interfaces/src/errors/not-found-error.ts
// etc.
```

**Verification:** `APICallError.isInstance(error)` works across bundle boundaries.

---

### 4. Set Up Project Structure with pnpm + Turbo

**Rationale:** Vercel AI SDK uses pnpm + Turborepo. This enables:
- Fast installs (pnpm)
- Task caching and parallelism (Turbo)
- Independent package versioning

**Action:**
1. Initialize pnpm workspace: `pnpm init`
2. Create `pnpm-workspace.yaml`
3. Create `turbo.json` with task pipeline
4. Set up TypeScript project references

**Verification:** `pnpm build` builds all packages. `pnpm --filter=core build` builds single package.

---

## P1: Core Functionality

### 5. Implement Provider Adapter Pattern

**Rationale:** Each AI provider implements the same interface. Adapters translate between internal types and provider API formats.

**Action:** Create `packages/providers/openai/src/`:

```typescript
// openai-language-model.ts
export class OpenAILLanguageModel implements LanguageModelV1 {
  readonly specificationVersion = 'v1';
  readonly provider = 'openai';
  readonly modelId: string;

  async doGenerate(options) {
    // 1. Convert options to OpenAI API format
    // 2. Make HTTP request
    // 3. Convert response to LanguageModelV1GenerateResult
  }

  doStream(options) { /* similar */ }
}

// openai-provider.ts
export function createOpenAI(settings?: OpenAIProviderSettings): OpenAIProvider {
  const provider = (modelId: string) => new OpenAILLanguageModel(modelId, settings);
  provider.specificationVersion = 'v1';
  provider.languageModel = provider;
  return provider as OpenAIProvider;
}
```

**Verification:** `createOpenAI()` returns a callable that produces a `LanguageModelV1`.

---

### 6. Build API Key Loading Pattern

**Rationale:** Vercel AI SDK's `loadApiKey` handles edge runtime, invalid types, and clear error messages.

**Action:** Create `packages/utils/src/load-api-key.ts`:

```typescript
export function loadApiKey({
  apiKey,
  environmentVariableName,
  description,
}: {
  apiKey: string | undefined;
  environmentVariableName: string;
  description: string;
}): string {
  // 1. Accept string apiKey if provided
  // 2. Reject non-string values with clear error
  // 3. Check environment variable
  // 4. Handle edge runtime (process undefined)
  // 5. Return key or throw LoadAPIKeyError
}
```

**Verification:** `loadApiKey({ apiKey: undefined, env: 'OPENAI_API_KEY' })` throws in Node; returns key in browser if passed directly.

---

### 7. Implement Retry with Exponential Backoff

**Rationale:** Network requests fail. Proper retry handling with provider-specific headers (retry-after, retry-after-ms) improves reliability.

**Action:** Create `packages/utils/src/retry.ts`:

```typescript
export async function retryWithExponentialBackoff<Output>(
  fn: () => Promise<Output>,
  options: {
    maxRetries?: number;
    baseDelayMs?: number;
    maxDelayMs?: number;
    retryable?: (error: APICallError) => boolean;
  }
): Promise<Output> {
  // Implement exponential backoff
  // Respect retry-after-ms header (OpenAI pattern)
  // Respect retry-after header (standard HTTP)
  // Don't retry on abort
  // Don't retry on non-retryable errors
}
```

**Verification:** Rate-limited API returns correct delays. Abort signal cancels retry loop.

---

### 8. Create User-Facing API Layer

**Rationale:** Core `generateText` function orchestrates model calls, retries, error handling, and response parsing.

**Action:** Create `packages/core/src/generate-text.ts`:

```typescript
export async function generateText(options: GenerateTextOptions): Promise<GenerateTextResult> {
  // 1. Resolve model (string -> LanguageModelV1)
  // 2. Validate inputs
  // 3. Call model.doGenerate with retry
  // 4. Parse response
  // 5. Return structured result
}
```

**Verification:** `generateText({ model: 'gpt-4o', prompt: 'Hello' })` returns text without streaming.

---

### 9. Implement Streaming Support

**Rationale:** Streaming is essential for AI UIs. Vercel AI SDK uses Web Streams API with TransformStreams.

**Action:** Create `packages/core/src/stream-text.ts`:

```typescript
export function streamText(options: StreamTextOptions): StreamTextResult {
  return {
    stream: createAsyncIterableStream(transformedStream),
    response,  // Metadata
    rawCall,    // For debugging
  };
}
```

**Transform stream pattern:**
```typescript
const transformedStream = originalStream
  .pipeThrough(convertToStandardFormat())
  .pipeThrough(handleToolCalls());
```

**Verification:** Streaming returns chunks immediately, not after completion.

---

### 10. Add Type Testing Infrastructure

**Rationale:** Vercel AI SDK catches type regressions with `*.test-d.ts` files. TypeScript compilation alone misses edge cases.

**Action:**
1. Install `vitest` and `expect-type`
2. Create test file pattern: `*.test-d.ts` co-located
3. Add to test script: `vitest --config vitest.config.ts`

```typescript
// Example: my-function.test-d.ts
import { describe, expectTypeOf } from 'vitest';

describe('MyFunction types', () => {
  it('should infer correct output type', () => {
    const result = myFunction({ input: 'test' });
    expectTypeOf(result).toEqualTypeOf<{ output: string }>();
  });
});
```

**Verification:** `pnpm test` runs both unit tests and type tests.

---

## P2: Enhancement

### 11. Implement Telemetry with Privacy Controls

**Rationale:** Observability is critical for production. Vercel AI SDK's implementation respects user privacy with `recordInputs`/`recordOutputs` flags.

**Action:** Create `packages/core/src/telemetry/`:

```typescript
// telemetry-settings.ts
export interface TelemetrySettings {
  isEnabled?: boolean;
  recordInputs?: boolean;   // Default: true
  recordOutputs?: boolean;  // Default: true
}

// Lazy evaluation pattern
export function selectAttributes(telemetry, attributes) {
  if (telemetry?.isEnabled !== true) {
    return {};  // Early return, no computation
  }
  // Only evaluate if recording enabled
}
```

**Verification:** With telemetry disabled, no attribute computation occurs.

---

### 12. Build Structured Output Support

**Rationale:** `generateObject` with Zod schema validation is a killer feature. Enables type-safe JSON generation.

**Action:** Create `packages/core/src/generate-object.ts`:

```typescript
export async function generateObject<Schema extends z.ZodSchema>({
  model,
  schema,
  prompt,
}: {
  model: LanguageModelV1;
  schema: Schema;
  prompt: string;
}): Promise<z.infer<Schema>> {
  // 1. Convert Zod schema to JSON Schema
  // 2. Request structured output from model
  // 3. Parse and validate response
  // 4. Return typed result
}
```

**Verification:** `generateObject({ model, schema: z.object({ name: z.string() }), prompt })` returns typed object.

---

### 13. Create Framework Integration Shell

**Rationale:** Vercel AI SDK supports React, Vue, Svelte, Angular. Framework-agnostic core enables this.

**Action:** Create `packages/react/src/` with React hooks:

```typescript
// Pattern to follow: AbstractChat + ChatState interface
// packages/core/src/ui/chat.ts - AbstractChat (framework-agnostic)
// packages/react/src/chat.ts - ReactChatState + useChat hook

export function useChat(options: UseChatOptions) {
  // Use useSyncExternalStore for reactivity
  // Callbacks ref pattern to avoid stale closures
  return { messages, sendMessage, status };
}
```

**Verification:** React app using `useChat` responds to state changes without re-render leaks.

---

### 14. Add Middleware System

**Rationale:** Cross-cutting concerns (logging, caching, defaults) without modifying core.

**Action:** Create `packages/core/src/middleware/`:

```typescript
// middleware.ts
export function wrapLanguageModel({ model, middleware }) {
  return [...middleware].reverse().reduce(
    (wrapped, mw) => doWrap({ model: wrapped, middleware: mw }),
    model
  );
}

// Example: default settings middleware
const defaultSettingsMiddleware = {
  transformParams: ({ params }) => ({
    ...params,
    temperature: params.temperature ?? 0.7,
  }),
};
```

**Verification:** Middleware transforms parameters before reaching model.

---

### 15. Implement Embeddings

**Rationale:** Vector embeddings are fundamental for RAG and similarity search. Vercel AI SDK has mature embedding support.

**Action:** Create `packages/core/src/embed.ts`:

```typescript
export async function embed(options: EmbedOptions): Promise<EmbedResult> {
  // Simple single-value embedding
}

export async function embedMany(options: EmbedManyOptions): Promise<EmbedManyResult> {
  // Batch with automatic chunking based on model's maxEmbeddingsPerCall
  // Configurable maxParallelCalls
}
```

**Verification:** `embedMany({ values: [...], model })` correctly batches large requests.

---

## P3: Future

### 16. Add Provider Registry

**Rationale:** Vercel AI SDK's `ProviderRegistry` enables provider lookup by string.

**Action:**
```typescript
interface ProviderRegistry {
  languageModel(modelId: string): LanguageModelV1;
  embeddingModel(modelId: string): EmbeddingModelV1;
  // etc.
}

function createRegistry(providers: Provider[]): ProviderRegistry {
  // Register all providers
  // Enable lookup by modelId string
}
```

**Verification:** `registry.languageModel('gpt-4o')` returns correct model.

---

### 17. Implement Tool Calling

**Rationale:** Tool calling enables agents and function execution. Vercel AI SDK's tool loop is a thin wrapper around `generateText`.

**Action:**
```typescript
export const tool = ({ name, description, parameters, execute }) => ({
  name,
  description,
  parameters: z.object(parameters),
  execute,
});

export async function executeToolCall({ toolCall, tools }) {
  // Find tool by name
  // Validate input against schema
  // Execute with timeout
  // Return result
}
```

**Verification:** `generateText({ tools: [weatherTool], prompt: 'Weather in NYC' })` executes weather tool and returns result.

---

### 18. Add Image Generation Support

**Rationale:** Multi-modal support (image generation, transcription, TTS) increases SDK value.

**Action:** Create `packages/core/src/generate-image.ts`:

```typescript
export async function generateImage({
  model,
  prompt,
  n = 1,
  size,
}: {
  model: ImageModelV1;
  prompt: string;
  n?: number;
  size?: string;
}): Promise<GenerateImageResult> {
  // Handle parallelization for n > maxImagesPerCall
  // Return GeneratedFile with data, mediaType, format
}
```

**Verification:** `generateImage({ model, prompt: 'A cat' })` returns base64 image data.

---

### 19. Dual Runtime Testing (Node + Edge)

**Rationale:** AI SDKs often run in Edge environments. Vercel AI SDK tests both.

**Action:**
```bash
pnpm test:node    # vitest (Node.js)
pnpm test:edge    # vitest with @edge-runtime/vm
```

**Verification:** Same tests pass on both runtimes.

---

### 20. Documentation and Examples

**Rationale:** SDK adoption depends on documentation quality. Vercel AI SDK has comprehensive docs at ai-sdk.dev.

**Action:**
1. Create API reference with type signatures
2. Write getting started guide
3. Add provider-specific examples
4. Document error codes and recovery patterns

**Verification:** New developer can make first API call in under 5 minutes following docs.

---

## Anti-Patterns to Avoid

Based on Vercel AI SDK's shortcomings:

| Anti-Pattern | Why Avoid | Prevention |
|--------------|-----------|------------|
| Extensive lint disabling | Inconsistent code | Minimal enforced rules |
| TODOs in code | Technical debt | GitHub issues instead |
| Deprecated APIs | Bloated surface | Sunset dates, major version removal |
| Inconsistent feature maturity | User confusion | Stabilize before marketing |
| No pre-commit type check | CI failures | Run tsc in pre-commit |

---

## Phased Implementation Order

**Phase 1 (Foundation):**
1. Project setup (pnpm + Turbo)
2. Interface definitions
3. Error hierarchy
4. Secure JSON parsing

**Phase 2 (Core):**
5. Provider adapter (OpenAI)
6. API key loading
7. Retry logic
8. generateText + streamText

**Phase 3 (Testing):**
9. Type testing setup
10. Unit tests
11. Integration tests

**Phase 4 (Enhancement):**
12. Telemetry
13. Structured output
14. Framework hooks (React)

**Phase 5 (Ecosystem):**
15. Additional providers
16. Tool calling
17. Image generation
18. Documentation

---

*These action items are derived from analyzing the Vercel AI SDK's architecture, code quality practices, and feature set. Adjust based on specific use case requirements and resource constraints.*
