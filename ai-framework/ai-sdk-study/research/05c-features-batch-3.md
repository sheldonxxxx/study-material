# Deep Dive: Embeddings, Speech/Audio Generation, Video Generation

## Overview

This batch covers three secondary features: **Embeddings** (CORE priority per index), **Speech/Audio Generation**, and **Video Generation** (both SECONDARY priority). The SDK shows clear maturity differences between these features, with embeddings being the most mature and well-integrated, while speech and video remain experimental with fewer providers and less polish.

---

## 1. Embeddings (`embed`, `embedMany`)

### API Surface

**Files:**
- `packages/ai/src/embed/embed.ts` - Single embedding function
- `packages/ai/src/embed/embed-many.ts` - Batch embedding function
- `packages/ai/src/embed/embed-result.ts` - Result interface
- `packages/ai/src/embed/embed-many-result.ts` - Batch result interface
- `packages/ai/src/embed/embed-events.ts` - Event types for callbacks
- `packages/provider/src/embedding-model/v4/embedding-model-v4.ts` - Provider interface

### Core Functions

**`embed(model, value)`** - Generates a single embedding vector from a string value.

**`embedMany(model, values, options?)`** - Generates embeddings for multiple values with intelligent chunking:
- Automatically splits large requests into smaller chunks if the model has `maxEmbeddingsPerCall` limit
- Supports `maxParallelCalls` for controlling concurrent API requests
- Handles `supportsParallelCalls` flag from the model to determine if parallel execution is safe

### Key Implementation Details

**Model Interface (EmbeddingModelV4):**
```typescript
interface EmbeddingModelV4 {
  specificationVersion: 'v4';
  provider: string;
  modelId: string;
  maxEmbeddingsPerCall: number | undefined | PromiseLike<number | undefined>;
  supportsParallelCalls: boolean | PromiseLike<boolean>;
  doEmbed(options: EmbeddingModelV4CallOptions): PromiseLike<EmbeddingModelV4Result>;
}
```

**Call Options:**
```typescript
interface EmbeddingModelV4CallOptions {
  values: Array<string>;  // Text values to embed
  abortSignal?: AbortSignal;
  providerOptions?: SharedV4ProviderOptions;
  headers?: SharedV4Headers;
}
```

**Result:**
```typescript
interface EmbedResult {
  value: string;
  embedding: Embedding;  // Float32Array or number[]
  usage: EmbeddingModelUsage;
  warnings: Array<Warning>;
  providerMetadata?: ProviderMetadata;
  response?: { headers?: Record<string, string>; body?: unknown };
}
```

### Features

| Feature | Status |
|---------|--------|
| Telemetry | Full support via `experimental_telemetry` |
| onStart callback | `experimental_onStart` |
| onFinish callback | `experimental_onFinish` |
| Retry logic | Automatic via `prepareRetries` with `maxRetries` |
| Provider metadata | Passed through from provider |
| Usage tracking | `EmbeddingModelUsage` with token counts |
| Warnings | Collected and returned in result |
| Chunking (embedMany) | Automatic based on `maxEmbeddingsPerCall` |
| Parallel calls | Configurable via `maxParallelCalls` |

### Provider Support

**~30+ providers support embeddings**, including:
- OpenAI (`text-embedding-3-small`, `text-embedding-3-large`, `text-embedding-ada-002`)
- Google (via `google-vertex`)
- Azure OpenAI
- Cohere
- Amazon Bedrock
- Fireworks
- Mistral
- TogetherAI
- And many more

### Polish Level

**HIGHEST** - Embeddings are very well-polished:
- Full telemetry integration
- Event callback system (onStart/onFinish)
- Automatic request chunking with parallel execution
- Comprehensive test coverage (embed.test.ts: 645 lines, embed-many.test.ts: 1091 lines)
- Usage tracking
- Warning collection

---

## 2. Speech/Audio Generation (`experimental_generateSpeech`)

### API Surface

**Files:**
- `packages/ai/src/generate-speech/generate-speech.ts` - Main function
- `packages/ai/src/generate-speech/generate-speech-result.ts` - Result interface
- `packages/ai/src/generate-speech/generated-audio-file.ts` - Audio file wrapper
- `packages/ai/src/generate-speech/index.ts` - Exports as `experimental_generateSpeech`
- `packages/provider/src/speech-model/v4/speech-model-v4.ts` - Provider interface

### Function Signature

```typescript
async function generateSpeech({
  model: SpeechModel,
  text: string,
  voice?: string,
  outputFormat?: 'mp3' | 'wav' | (string & {}),
  instructions?: string,  // e.g., "Speak in a slow and steady tone"
  speed?: number,
  language?: string,  // ISO 639-1 or "auto"
  providerOptions?: ProviderOptions,
  maxRetries?: number,
  abortSignal?: AbortSignal,
  headers?: Record<string, string>,
}): Promise<SpeechResult>
```

### Result

```typescript
interface SpeechResult {
  audio: GeneratedAudioFile;  // Has data, mediaType, format
  warnings: Array<Warning>;
  responses: Array<SpeechModelResponseMetadata>;
  providerMetadata: Record<string, JSONObject>;
}
```

### Key Implementation Details

**Speech Model Interface:**
```typescript
interface SpeechModelV4 {
  specificationVersion: 'v4';
  provider: string;
  modelId: string;
  doGenerate(options: SpeechModelV4CallOptions): PromiseLike<{
    audio: string | Uint8Array;  // Raw audio data
    warnings: Array<SharedV4Warning>;
    request?: { body?: unknown };
    response: {
      timestamp: Date;
      modelId: string;
      headers?: SharedV2Headers;
      body?: unknown;
    };
    providerMetadata?: Record<string, JSONObject>;
  }>;
}
```

**Error Handling:**
- `NoSpeechGeneratedError` - thrown when no audio is returned

**Audio File Handling:**
- Media type detection via `detectMediaType` with `audioMediaTypeSignatures`
- Format extraction from media type (mp3, wav, etc.)
- `DefaultGeneratedAudioFile` extends `GeneratedFile` with format property

### Features

| Feature | Status |
|---------|--------|
| Telemetry | **NOT SUPPORTED** |
| onStart callback | **NOT SUPPORTED** |
| onFinish callback | **NOT SUPPORTED** |
| Retry logic | Supported via `prepareRetries` |
| Provider metadata | Supported |
| Usage tracking | Not in result |
| Warnings | Supported |
| Media type detection | Automatic via magic bytes |

### Provider Support

**~6 providers** support speech generation:
- OpenAI (via `openai` provider)
- ElevenLabs (`elevenlabs` provider)
- LMNT (`lmnt` provider)
- Hume (`hume` provider)
- Deepgram (`deepgram` provider - though primarily transcription)
- Fal (`fal` provider)

### Polish Level

**MEDIUM** - Speech is functional but less polished than embeddings:
- No telemetry integration
- No event callbacks (onStart/onFinish)
- Basic error handling with `NoSpeechGeneratedError`
- Moderate test coverage (generate-speech.test.ts: 300 lines)
- Still marked `experimental_` in exports

---

## 3. Video Generation (`experimental_generateVideo`)

### API Surface

**Files:**
- `packages/ai/src/generate-video/generate-video.ts` - Main function
- `packages/ai/src/generate-video/generate-video-result.ts` - Result interface
- `packages/ai/src/generate-video/index.ts` - Exports as `experimental_generateVideo`
- `packages/provider/src/video-model/v4/video-model-v4.ts` - Provider interface

### Function Signature

```typescript
async function experimental_generateVideo({
  model: VideoModel,
  prompt: GenerateVideoPrompt,  // string | { image?: DataContent; text?: string }
  n?: number,  // Number of videos to generate
  maxVideosPerCall?: number,
  aspectRatio?: `${number}:${number}`,  // e.g., "16:9"
  resolution?: `${number}x${number}`,   // e.g., "1280x720"
  duration?: number,  // seconds
  fps?: number,
  seed?: number,
  providerOptions?: ProviderOptions,
  maxRetries?: number,
  abortSignal?: AbortSignal,
  headers?: Record<string, string>,
  download?: (options) => Promise<{ data: Uint8Array; mediaType: string | undefined }>,
}): Promise<GenerateVideoResult>
```

### Result

```typescript
interface GenerateVideoResult {
  video: GeneratedFile;        // First video
  videos: Array<GeneratedFile>; // All videos
  warnings: Array<Warning>;
  responses: Array<VideoModelResponseMetadata>;
  providerMetadata: VideoModelProviderMetadata;
}
```

### Key Implementation Details

**Video Model Interface:**
```typescript
interface VideoModelV4 {
  specificationVersion: 'v4';
  provider: string;
  modelId: string;
  maxVideosPerCall: number | undefined | GetMaxVideosPerCallFunction;
  doGenerate(options: VideoModelV4CallOptions): PromiseLike<{
    videos: Array<VideoModelV4VideoData>;  // url | base64 | binary
    warnings: Array<SharedV4Warning>;
    providerMetadata?: SharedV4ProviderMetadata;
    response: {
      timestamp: Date;
      modelId: string;
      headers?: Record<string, string>;
    };
  }>;
}
```

**Prompt Types:**
```typescript
type GenerateVideoPrompt =
  | string
  | {
      image: DataContent;  // Base64, Uint8Array, or URL
      text?: string;
    };
```

**Video Data Handling:**
- Supports three video data types: `url`, `base64`, `binary`
- URL videos are automatically downloaded via `createDownload()` (2 GiB limit)
- Media type detection via `detectMediaType` with `videoMediaTypeSignatures`
- Fallback to `video/mp4` if detection fails

**Parallelization:**
- Automatically parallelizes multiple video requests via `Promise.all`
- Calculates `callCount` based on `n` and `maxVideosPerCall`
- Each batch calls `model.doGenerate` once for `n` videos

### Features

| Feature | Status |
|---------|--------|
| Telemetry | **NOT SUPPORTED** |
| onStart callback | **NOT SUPPORTED** |
| onFinish callback | **NOT SUPPORTED** |
| Retry logic | Supported via `prepareRetries` |
| Provider metadata | Supported |
| Usage tracking | Not in result |
| Warnings | Supported |
| Media type detection | Automatic via magic bytes |
| Custom download | Supported via `download` option |
| Image-to-video | Supported via prompt.image |

### Provider Support

**~9 providers** support video generation:
- KlingAI (`klingai` provider)
- Google (`google` provider - Veo)
- Google Vertex (`google-vertex` provider)
- xAI (`xai` provider - Grok)
- Alibaba (`alibaba` provider - Wan)
- Replicate (`replicate` provider)
- Fal (`fal` provider)
- ByteDance (`bytedance` provider)
- Prodia (`prodia` provider)

### Polish Level

**MEDIUM-LOW** - Video is functional but least polished:
- No telemetry integration
- No event callbacks
- Error handling with `NoVideoGeneratedError` (has deprecated methods)
- Good test coverage but feature still experimental (generate-video.test.ts: 982 lines)
- Still marked `experimental_` in exports
- Note: `NoVideoGeneratedError` has deprecated `isNoVideoGeneratedError` method and `toJSON` - suggests older pattern

---

## Comparison: Polish vs. Core Features

### Embeddings vs Speech vs Video

| Aspect | Embeddings | Speech | Video |
|--------|------------|--------|-------|
| **Priority (Index)** | CORE | SECONDARY | SECONDARY |
| **Export name** | `embed` | `experimental_generateSpeech` | `experimental_generateVideo` |
| **Telemetry** | Full | None | None |
| **Callbacks** | onStart, onFinish | None | None |
| **Providers** | 30+ | ~6 | ~9 |
| **Test lines** | 1736 (embed + embed-many) | 300 | 982 |
| **Usage tracking** | Yes | No | No |
| **Chunking** | Automatic (embedMany) | N/A | Automatic batching |
| **Parallelization** | Configurable | N/A | Automatic |

### Comparison with Core Text Generation

Text generation (`generateText`, `streamText`) is the most mature feature with:
- Full telemetry
- Streaming support
- Tool calling
- Agent framework integration
- 40+ providers
- Extensive middleware support

Embeddings come second in maturity, with good infrastructure but fewer lifecycle features than text.

Speech and Video are the least polished secondary features, remaining experimental with limited provider support and no telemetry/callback infrastructure.

---

## Architecture Observations

### Provider Integration Pattern

All three features follow the same pattern:
1. User calls SDK function (`embed`, `generateSpeech`, `experimental_generateVideo`)
2. SDK resolves model via `resolveEmbeddingModel`, `resolveSpeechModel`, `resolveVideoModel`
3. SDK calls `model.doEmbed()` or `model.doGenerate()`
4. SDK wraps result in appropriate result class

### Key Files for Model Resolution

- `packages/ai/src/model/resolve-model.ts` - Contains all model resolvers
- `packages/ai/src/types/video-model.ts` - VideoModel type (union of v3 and v4)
- `packages/ai/src/types/speech-model.ts` - SpeechModel type

### Important Note: Video is NOT in ProviderRegistry

Unlike `embeddingModel`, `imageModel`, `speechModel`, etc., the `videoModel` is **NOT** part of the `ProviderRegistryProvider` interface in `packages/ai/src/registry/provider-registry.ts`. Video models are only accessible via direct provider methods (e.g., `klingai.videoModel('model-id')`).

This suggests video generation may be at an earlier stage of integration or considered more provider-specific.

---

## Summary

**Embeddings** are well-engineered with automatic chunking, parallel execution control, telemetry, and comprehensive result types. They work across 30+ providers and represent the SDK's most mature secondary feature.

**Speech generation** is functional but basic - no telemetry, no callbacks, limited providers (~6). It handles the core TTS use case but lacks the observability infrastructure of embeddings.

**Video generation** is the least polished - also missing telemetry/callbacks, only ~9 providers, and error handling shows signs of older patterns (deprecated methods). However, it has good test coverage (982 lines) and supports interesting features like image-to-video and automatic download of URL-based videos.

The gap between core features (text generation) and secondary features (embeddings, speech, video) is significant, with speech and video lacking telemetry and callback infrastructure that embeddings have.
