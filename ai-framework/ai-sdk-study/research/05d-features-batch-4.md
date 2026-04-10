# Feature Deep Dive: Speech-to-Text, Reranking, Telemetry

## Overview

This document examines three secondary features in the Vercel AI SDK: Speech-to-Text/Transcription, Reranking, and Telemetry & Observability. These are categorized as SECONDARY priority in the feature index, distinguishing them from CORE features like text generation and embeddings.

---

## 1. Speech-to-Text / Transcription

### Implementation Location
`packages/ai/src/transcribe/`

### Core Files
- `transcribe.ts` - Main implementation (187 lines)
- `transcribe-result.ts` - Result interface definition
- `index.ts` - Exports with `experimental_` prefix

### API Design

```typescript
// Usage pattern
const result = await transcribe({
  model: deepgram.transcriptionModel('nova-2'),
  audio: audioData, // DataContent or URL
  providerOptions: { deepgram: { language: 'en' } },
  maxRetries: 2,
  abortSignal,
});
```

### Key Characteristics

**Model Resolution:**
- Uses `resolveTranscriptionModel()` from `model/resolve-model.ts`
- Accepts `string | TranscriptionModelV4 | TranscriptionModelV3 | TranscriptionModelV2`
- Falls back to global provider's `transcriptionModel()` method

**Input Handling:**
- Accepts `DataContent` (string | Uint8Array | ArrayBuffer | Buffer) or `URL`
- URL audio is downloaded via `createDownload()` utility (2 GiB default limit)
- Media type detection via `detectMediaType()` with `audioMediaTypeSignatures`

**Result Structure:**
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

### Provider Interface (`@ai-sdk/provider`)

Located in `transcription-model/v4/transcription-model-v4.ts`:

```typescript
export type TranscriptionModelV4 = {
  readonly specificationVersion: 'v4';
  readonly provider: string;
  readonly modelId: string;
  doGenerate(options: TranscriptionModelV4CallOptions): PromiseLike<{
    text: string;
    segments: Array<{ text: string; startSecond: number; endSecond: number }>;
    language: string | undefined;
    durationInSeconds: number | undefined;
    warnings: Array<SharedV4Warning>;
    response: { timestamp: Date; modelId: string; headers?: SharedV4Headers; body?: unknown };
    providerMetadata?: Record<string, JSONObject>;
  }>;
};
```

### Provider Implementations

**Transcription Providers (9):**
| Provider | Package | Notable Features |
|----------|---------|------------------|
| Deepgram | `packages/deepgram/` | diarize, smart formatting, summarization, topic/intent detection |
| OpenAI | `packages/openai/` | Whisper-based, timestamp granularities |
| Rev AI | `packages/revai/` | Speaker diarization, custom vocabulary |
| Groq | `packages/groq/` | Fast inference |
| Gladia | `packages/gladia/` | Language detection |
| ElevenLabs | `packages/elevenlabs/` | Voice cloning support |
| AssemblyAI | `packages/assemblyai/` | Real-time, lemmatization |
| TogetherAI | `packages/togetherai/` | Multi-modal |
| KlingAI | `packages/klingai/` | Video-related audio |

### Provider Implementation Example (Deepgram)

The Deepgram implementation (`deepgram-transcription-model.ts`) demonstrates the pattern:
- Uses `postToApi()` from provider-utils for HTTP requests
- Builds query parameters (not JSON body) for Deepgram API
- Provider options parsed via `parseProviderOptions()` with Zod schema
- Response parsed using `createJsonResponseHandler()` with Zod schema

### Export Pattern

```typescript
// index.ts exports with experimental prefix
export { transcribe as experimental_transcribe } from './transcribe';
export type { TranscriptionResult as Experimental_TranscriptionResult } from './transcribe-result';
```

---

## 2. Reranking

### Implementation Location
`packages/ai/src/rerank/`

### Core Files
- `rerank.ts` - Main implementation (349 lines)
- `rerank-result.ts` - Result interface
- `rerank-events.ts` - Event types for callbacks
- `index.ts` - Exports

### Key Characteristics

**Generic Document Support:**
```typescript
// Reranks both text and object documents
export async function rerank<VALUE extends JSONObject | string>({
  model,
  documents,
  query,
  topN,
  maxRetries,
  abortSignal,
  headers,
  providerOptions,
  experimental_telemetry: telemetry,
  experimental_onStart: onStart,
  experimental_onFinish: onFinish,
}): Promise<RerankResult<VALUE>>
```

**Document Type Handling:**
```typescript
// Internal conversion to provider format
const documentsToSend: RerankingModelV4CallOptions['documents'] =
  typeof documents[0] === 'string'
    ? { type: 'text', values: documents as string[] }
    : { type: 'object', values: documents as JSONObject[] };
```

**Event System:**
```typescript
// Rerank-specific events (4 event types)
export interface RerankOnStartEvent { /* callId, operationId, provider, modelId, documents, query, topN, etc. */ }
export interface RerankOnFinishEvent { /* + ranking, warnings, providerMetadata, response */ }
export interface RerankStartEvent { /* Inner doRerank start */ }
export interface RerankFinishEvent { /* Inner doRerank finish */ }
```

**Telemetry Integration:**
- Rerank has first-class telemetry support
- Uses `getGlobalTelemetryIntegration()` like generateText/embed
- Supports `experimental_telemetry` parameter
- Telemetry spans track: `ai.rerank` (root), `ai.rerank.doRerank` (inner)

### Result Structure

```typescript
export interface RerankResult<VALUE> {
  readonly originalDocuments: Array<VALUE>;
  readonly rerankedDocuments: Array<VALUE>; // Sorted by score desc
  readonly ranking: Array<{
    originalIndex: number;
    score: number;
    document: VALUE;
  }>;
  readonly providerMetadata?: ProviderMetadata;
  readonly response: {
    id?: string;
    timestamp: Date;
    modelId: string;
    headers?: Record<string, string>;
    body?: unknown;
  };
}
```

### Provider Interface (`@ai-sdk/provider`)

Located in `reranking-model/v4/reranking-model-v4.ts`:

```typescript
export type RerankingModelV4 = {
  readonly specificationVersion: 'v4';
  readonly provider: string;
  readonly modelId: string;
  doRerank(options: RerankingModelV4CallOptions): PromiseLike<{
    ranking: Array<{ index: number; relevanceScore: number }>;
    providerMetadata?: SharedV4ProviderMetadata;
    warnings?: Array<SharedV4Warning>;
    response?: {
      id?: string;
      timestamp?: Date;
      modelId?: string;
      headers?: SharedV4Headers;
      body?: unknown;
    };
  }>;
};
```

### Provider Implementations

**Reranking Providers (2):**
| Provider | Package | Document Types |
|----------|---------|----------------|
| Cohere | `packages/cohere/` | text, object (converted to JSON string) |
| TogetherAI | `packages/togetherai/` | text, object |

### Cohere Implementation Pattern

```typescript
// Documents converted based on type
const documentsToSend =
  documents.type === 'text'
    ? documents.values
    : documents.values.map(value => JSON.stringify(value));

// API call
await postJsonToApi({
  url: `${this.config.baseURL}/rerank`,
  body: {
    model: this.modelId,
    query,
    documents: documentsToSend,
    top_n: topN,
    max_tokens_per_doc: rerankingOptions?.maxTokensPerDoc,
    priority: rerankingOptions?.priority,
  },
});
```

### Empty Document Edge Case

Rerank handles empty documents as a special case:
```typescript
if (documents.length === 0) {
  // Still fires onStart/onFinish events with empty ranking
  // Returns DefaultRerankResult with originalDocuments: [], ranking: []
}
```

---

## 3. Telemetry & Observability

### Implementation Location
`packages/ai/src/telemetry/`

### Core Files
- `open-telemetry-integration.ts` - OpenTelemetry implementation (875 lines)
- `telemetry-integration.ts` - Integration interface
- `telemetry-settings.ts` - Configuration types
- `telemetry-integration-registry.ts` - Global registry
- `get-global-telemetry-integration.ts` - Integration factory
- `select-telemetry-attributes.ts` - Attribute filtering
- `record-span.ts` - Span recording utilities
- `assemble-operation-name.ts` - Operation naming
- `get-base-telemetry-attributes.ts` - Base attribute generation
- `stringify-for-telemetry.ts` - Safe JSON serialization
- `noop-tracer.ts` - No-op implementation
- `get-tracer.ts` - Tracer retrieval

### Architecture

```
TelemetrySettings (per-call config)
         |
         v
getGlobalTelemetryIntegration() --> TelemetryIntegrationRegistry (global)
         |                                    |
         v                                    v
   CompositeIntegration              registerTelemetryIntegration()
         |
         v
OpenTelemetryIntegration + Custom Integrations
```

### TelemetrySettings Configuration

```typescript
export type TelemetrySettings = {
  isEnabled?: boolean;           // Enable/disable (disabled by default experimental)
  recordInputs?: boolean;         // Default: true
  recordOutputs?: boolean;        // Default: true
  functionId?: string;           // Grouping identifier
  metadata?: Record<string, JSONValue>;
  tracer?: Tracer;                // Custom OTel tracer
  integrations?: TelemetryIntegration | TelemetryIntegration[];
};
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

### OpenTelemetryIntegration Implementation

**Key Design Patterns:**

1. **Call State Management:**
   ```typescript
   interface CallState {
     operationId: string;
     telemetry: TelemetrySettings | undefined;
     rootSpan: Span | undefined;
     rootContext: Context | undefined;
     stepSpan: Span | undefined;
     stepContext: Context | undefined;
     embedSpans: Map<string, { span: Span; context: Context }>;
     rerankSpan: { span: Span; context: Context } | undefined;
     toolSpans: Map<string, { span: Span; context: Context }>;
     baseTelemetryAttributes: Attributes;
     settings: Record<string, unknown>;
   }
   ```

2. **Operation Routing:**
   ```typescript
   onStart(event: OnStartEvent | EmbedOnStartEvent | RerankOnStartEvent) {
     if (event.operationId === 'ai.embed' || event.operationId === 'ai.embedMany') {
       this.onEmbedOperationStart(event);
       return;
     }
     if (event.operationId === 'ai.rerank') {
       this.onRerankOperationStart(event);
       return;
     }
     this.onGenerateStart(event);
   }
   ```

3. **Attribute Selection with Lazy Evaluation:**
   ```typescript
   selectAttributes(telemetry, {
     'ai.prompt': {
       input: () => JSON.stringify({ system: event.system, prompt: event.prompt, messages: event.messages }),
     },
   });
   // Only evaluated if recordInputs !== false
   ```

4. **Context Propagation:**
   ```typescript
   executeTool({ callId, toolCallId, execute }) {
     const toolSpanEntry = this.getCallState(callId)?.toolSpans.get(toolCallId);
     if (toolSpanEntry == null) return execute();
     return context.with(toolSpanEntry.context, execute);
   }
   ```

### Traced Operations

| Operation | Span Name | Attributes |
|-----------|-----------|------------|
| generateText | `ai.generateText` | model, prompt, settings |
| streamText | `ai.streamText` | model, prompt, settings |
| generateText.doGenerate | `ai.generateText.doGenerate` | prompt messages, tools, usage |
| streamText.doStream | `ai.streamText.doStream` | prompt messages, tools, usage |
| ai.toolCall | `ai.toolCall` | tool name, args, result |
| embed | `ai.embed` / `ai.embedMany` | values |
| embed.doEmbed | `ai.embed.doEmbed` | embeddings, usage |
| rerank | `ai.rerank` | documents, query |
| rerank.doRerank | `ai.rerank.doRerank` | documents, ranking |

### Standardized Attribute Names

The implementation uses both AI SDK conventions and GenAI conventions:

**AI SDK Attributes:**
- `ai.model.provider`, `ai.model.id`
- `ai.prompt`, `ai.prompt.messages`, `ai.prompt.tools`, `ai.prompt.toolChoice`
- `ai.response.finishReason`, `ai.response.text`, `ai.response.toolCalls`, `ai.response.id`, `ai.response.timestamp`
- `ai.usage.inputTokens`, `ai.usage.outputTokens`, `ai.usage.totalTokens`
- `ai.toolCall.name`, `ai.toolCall.args`, `ai.toolCall.result`

**GenAI Convention Attributes (OpenTelemetry):**
- `gen_ai.system`, `gen_ai.request.model`
- `gen_ai.request.frequency_penalty`, `gen_ai.request.max_tokens`, `gen_ai.request.presence_penalty`
- `gen_ai.request.stop_sequences`, `gen_ai.request.temperature`, `gen_ai.request.top_k`, `gen_ai.request.top_p`
- `gen_ai.response.finish_reasons`, `gen_ai.response.id`, `gen_ai.response.model`
- `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`

### Global Registry Pattern

```typescript
// Register globally once
export function registerTelemetryIntegration(integration: TelemetryIntegration) {
  if (!globalThis.AI_SDK_TELEMETRY_INTEGRATIONS) {
    globalThis.AI_SDK_TELEMETRY_INTEGRATIONS = [];
  }
  globalThis.AI_SDK_TELEMETRY_INTEGRATIONS.push(integration);
}

// Composite integration factory
export function getGlobalTelemetryIntegration() {
  return ({ integrations } = {}) => {
    const allIntegrations = [...globalIntegrations, ...localIntegrations];
    return {
      onStart: createTelemetryComposite(integration => integration.onStart),
      // ... other methods
    };
  };
}
```

---

## Comparative Analysis: Secondary vs Core Features

### Similarities

1. **Same Architecture:** All features use the provider pattern with interface versioning (v2, v3, v4)
2. **Same Utilities:** Use `@ai-sdk/provider-utils` for HTTP calls, parsing, headers
3. **Same Error Handling:** Extend `AISDKError` with marker pattern
4. **Same Type System:** Zod for validation, shared types from `@ai-sdk/provider`
5. **Same Export Pattern:** Index re-exports, experimental prefix for newer features

### Differences

| Aspect | Core Features | Secondary Features |
|--------|---------------|-------------------|
| File Count | 54 subdirs (generate-text) | 4-6 files per feature |
| Provider Support | 33 providers | 2-9 providers |
| Event System | Rich multi-event (onStart, onChunk, onFinish, etc.) | Simplified or none |
| Telemetry | Full integration | Rerank has full, Transcription minimal |
| Streaming | Supported | Not applicable (Transcription is request-response) |
| Tool Support | Yes | No |

### Transcription-Specific Patterns

1. **No Streaming:** Transcription is purely request-response (audio in, text out)
2. **Media Type Detection:** Automatic detection of audio format using magic bytes
3. **Download Abstraction:** `createDownload()` for URL-based audio with size limits
4. **Segment Timing:** Word-level timing information preserved

### Reranking-Specific Patterns

1. **Generic Types:** Supports both text and JSON object documents
2. **Dual Event System:** Outer (ai.rerank) and inner (ai.rerank.doRerank) operations
3. **Empty Document Handling:** Graceful handling of empty document lists
4. **Score-based Ranking:** Results sorted by relevance score descending

### Telemetry-Specific Patterns

1. **Composition Pattern:** Multiple integrations combined into single composite
2. **Context Propagation:** OpenTelemetry context used for nested spans
3. **Lazy Attribute Evaluation:** Input/output attributes only computed when recording enabled
4. **Discriminated Union Events:** Event types distinguished by `operationId`

---

## Key Files Reference

### Transcription
- `packages/ai/src/transcribe/transcribe.ts` - Main API
- `packages/ai/src/transcribe/transcribe-result.ts` - Result interface
- `packages/provider/src/transcription-model/v4/transcription-model-v4.ts` - Provider interface
- `packages/deepgram/src/deepgram-transcription-model.ts` - Provider example
- `packages/openai/src/transcription/openai-transcription-model.ts` - Provider example

### Reranking
- `packages/ai/src/rerank/rerank.ts` - Main API
- `packages/ai/src/rerank/rerank-result.ts` - Result interface
- `packages/ai/src/rerank/rerank-events.ts` - Event types
- `packages/provider/src/reranking-model/v4/reranking-model-v4.ts` - Provider interface
- `packages/cohere/src/reranking/cohere-reranking-model.ts` - Provider example

### Telemetry
- `packages/ai/src/telemetry/open-telemetry-integration.ts` - OTel implementation
- `packages/ai/src/telemetry/telemetry-integration.ts` - Interface
- `packages/ai/src/telemetry/telemetry-settings.ts` - Configuration
- `packages/ai/src/telemetry/telemetry-integration-registry.ts` - Global registry
- `packages/ai/src/telemetry/get-global-telemetry-integration.ts` - Factory
