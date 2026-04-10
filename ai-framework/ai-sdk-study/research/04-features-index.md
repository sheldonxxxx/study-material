# Vercel AI SDK - Feature Index

## Overview

The AI SDK is a provider-agnostic TypeScript toolkit for building AI-powered applications. This index maps documented features to their implementation locations.

---

## Core Features

### 1. Text Generation
**Description:** Generate text completions with `generateText` and stream responses with `streamText`. Supports prompts, system messages, and multi-turn conversations.

**Key Files/Locations:**
- `packages/ai/src/generate-text/` (54 subdirectories - main implementation)
- `packages/ai/src/prompt/` (prompt construction)
- `packages/ai/src/types/` (type definitions)

**Priority:** CORE

---

### 2. Structured Object Generation
**Description:** Generate type-safe structured data (JSON) with schema validation using Zod. Features `generateObject` and `streamObject` with automatic schema inference.

**Key Files/Locations:**
- `packages/ai/src/generate-object/`
- `packages/ai/src/util/` (JSON parsing utilities)

**Priority:** CORE

---

### 3. Agent Framework
**Description:** Build AI agents with tool-calling capabilities using `ToolLoopAgent`. Supports custom tools, sandboxed shell execution, and multi-turn reasoning loops.

**Key Files/Locations:**
- `packages/ai/src/agent/`
- `packages/ai/src/middleware/` (agent middleware)
- `packages/ai/src/ui-message-stream/` (agent UI streaming)

**Priority:** CORE

---

### 4. UI Integration / Chat Hooks
**Description:** Framework-agnostic hooks for building chatbots and generative UIs: `useChat`, `useCompletion`, `useObject`. Available for React, Svelte, Vue, Angular, and Next.js RSC.

**Key Files/Locations:**
- `packages/react/src/` (useChat, useCompletion, useObject)
- `packages/svelte/src/`
- `packages/vue/src/`
- `packages/angular/src/`
- `packages/rsc/src/` (React Server Components)
- `packages/ai/src/ui/` (shared UI components)

**Priority:** CORE

---

### 5. Unified Provider Architecture
**Description:** Single API interface across 40+ AI providers (OpenAI, Anthropic, Google, Azure, Amazon Bedrock, Cohere, etc.). Uses Vercel AI Gateway for out-of-the-box access.

**Key Files/Locations:**
- `packages/provider/` (provider interface specifications)
- `packages/provider-utils/` (shared utilities)
- `packages/openai/`, `packages/anthropic/`, `packages/google/`, etc. (individual providers)
- `packages/gateway/` (Vercel AI Gateway integration)

**Priority:** CORE

---

### 6. Image Generation
**Description:** Generate images from text prompts with support for partial image streams and base64 output.

**Key Files/Locations:**
- `packages/ai/src/generate-image/`
- Provider packages with image generation support (e.g., `packages/openai/`)

**Priority:** CORE

---

### 7. Embeddings
**Description:** Generate vector embeddings for documents with `embed` and batch embedding with `embedMany`.

**Key Files/Locations:**
- `packages/ai/src/embed/`
- Provider packages with embedding support

**Priority:** CORE

---

## Secondary Features

### 8. Speech/Audio Generation
**Description:** Generate speech/audio from text (text-to-speech).

**Key Files/Locations:**
- `packages/ai/src/generate-speech/`
- Provider packages with TTS support

**Priority:** SECONDARY

---

### 9. Video Generation
**Description:** Generate video from text prompts.

**Key Files/Locations:**
- `packages/ai/src/generate-video/`
- Provider packages with video support (e.g., `packages/klingai/`)

**Priority:** SECONDARY

---

### 10. Speech-to-Text / Transcription
**Description:** Transcribe audio to text.

**Key Files/Locations:**
- `packages/ai/src/transcribe/`
- Provider packages with transcription (e.g., `packages/deepgram/`)

**Priority:** SECONDARY

---

### 11. Reranking
**Description:** Rerank search results for improved relevance.

**Key Files/Locations:**
- `packages/ai/src/rerank/`
- Provider packages with reranking (e.g., `packages/cohere/`)

**Priority:** SECONDARY

---

### 12. Telemetry & Observability
**Description:** Built-in telemetry support for tracing and monitoring AI calls.

**Key Files/Locations:**
- `packages/ai/src/telemetry/`
- Example: `examples/next-openai-telemetry/`
- Example: `examples/next-openai-telemetry-sentry/`

**Priority:** SECONDARY

---

## Implementation Locations Summary

| Feature Area | Main Package(s) | Type |
|-------------|-----------------|------|
| Text Generation | `packages/ai` | Core |
| Object Generation | `packages/ai` | Core |
| Agents | `packages/ai` | Core |
| UI Hooks | `packages/react`, `packages/svelte`, `packages/vue`, `packages/angular`, `packages/rsc` | Core |
| Providers | `packages/provider`, `packages/<provider-*>` | Core |
| Image Generation | `packages/ai` | Core |
| Embeddings | `packages/ai` | Core |
| Speech Generation | `packages/ai` | Secondary |
| Video Generation | `packages/ai` | Secondary |
| Transcription | `packages/ai` | Secondary |
| Reranking | `packages/ai` | Secondary |
| Telemetry | `packages/ai` | Secondary |

---

## Provider Ecosystem (40+)

The SDK supports the following provider categories:

**Major Providers:**
- OpenAI (GPT models)
- Anthropic (Claude models)
- Google (Gemini models)
- Azure (OpenAI on Azure)
- Amazon Bedrock (Claude, Llama, Titan)
- Cohere
- Hugging Face
- Mistral

**Specialized Providers:**
- AssemblyAI (speech)
- Deepgram (transcription)
- ElevenLabs (voice)
- Replicate (image/video models)
- Black Forest Labs (FLUX image models)
- TogetherAI
- Perplexity
- Groq
- Fireworks AI

**Gateway & Infrastructure:**
- Vercel AI Gateway (unified gateway)
- OpenAI-Compatible APIs
