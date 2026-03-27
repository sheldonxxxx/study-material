# memory-lancedb-pro Feature Index

## Overview

This document provides a prioritized feature index for `memory-lancedb-pro`, an OpenClaw plugin that provides AI long-term memory capabilities via LanceDB with hybrid retrieval.

---

## Core Features (Tier 1)

### 1. Hybrid Retrieval (Vector + BM25)
**Description:** Combines semantic vector similarity search with BM25 full-text keyword matching for recall that is both contextually aware and keyword-exact.

**Key Files:**
- `src/retriever.ts` (46KB) - Hybrid retrieval engine
- `src/store.ts` (36KB) - LanceDB storage with FTS indexing

**Priority:** CORE

---

### 2. Cross-Encoder Reranking
**Description:** Post-retrieval reranking using cross-encoder models from Jina, SiliconFlow, Voyage AI, or Pinecone. Hybrid scoring: 60% cross-encoder + 40% original fused score.

**Key Files:**
- `src/retriever.ts` - reranking logic
- `src/embedder.ts` (33KB) - embedding abstraction

**Priority:** CORE

---

### 3. Multi-Scope Isolation
**Description:** Memory isolation across `global`, `agent:<id>`, `project:<id>`, `user:<id>`, and `custom:<name>` scopes with configurable agent access control.

**Key Files:**
- `src/scopes.ts` (18KB) - scope management
- `src/clawteam-scope.ts` - team-level scope handling

**Priority:** CORE

---

### 4. Smart Memory Extraction
**Description:** LLM-powered extraction into 6 semantic categories (profile, preferences, entities, events, cases, patterns) with L0/L1/L2 layered storage and two-stage deduplication.

**Key Files:**
- `src/smart-extractor.ts` (45KB) - main extraction logic
- `src/extraction-prompts.ts` (8KB) - LLM prompts
- `src/smart-metadata.ts` (21KB) - metadata handling

**Priority:** CORE

---

### 5. Memory Lifecycle Management
**Description:** Weibull decay engine with composite scoring (recency + frequency + importance) and three-tier promotion system (Peripheral, Working, Core).

**Key Files:**
- `src/decay-engine.ts` (7KB) - Weibull decay model
- `src/tier-manager.ts` (5KB) - tier promotion/demotion
- `src/access-tracker.ts` (10KB) - access frequency tracking

**Priority:** CORE

---

### 6. Auto-Capture & Auto-Recall
**Description:** Automatic memory extraction on `agent_end` hook and context injection via `before_prompt_build` hook. Filters noise and adaptively skips retrieval for non-memory queries.

**Key Files:**
- `src/noise-filter.ts` (3KB) - content filtering
- `src/adaptive-retrieval.ts` (4KB) - retrieval decision
- `src/auto-capture-cleanup.ts` (3KB) - text normalization

**Priority:** CORE

---

## Secondary Features (Tier 2)

### 7. Management CLI
**Description:** Full CLI toolkit via `openclaw memory-pro` for list, search, stats, delete, delete-bulk, export, import, reembed, upgrade, migrate, and auth commands.

**Key Files:**
- `cli.ts` (47KB) - CLI implementation
- `src/tools.ts` (74KB) - MCP tools definition

**Priority:** SECONDARY

---

### 8. Multi-Provider Embedding
**Description:** Compatible with any OpenAI-compatible API provider including Jina, OpenAI, Voyage, Google Gemini, and Ollama (local).

**Key Files:**
- `src/embedder.ts` (33KB) - embedding abstraction layer
- `src/llm-client.ts` (13KB) - LLM API client

**Priority:** SECONDARY

---

### 9. Session Memory
**Description:** Session summarization on `/new` command with configurable message count. Stores session summaries to LanceDB for cross-session continuity.

**Key Files:**
- `src/session-compressor.ts` (10KB) - session compression
- `src/session-recovery.ts` (5KB) - recovery logic

**Priority:** SECONDARY

---

### 10. Workspace Boundary & Privacy
**Description:** Privacy controls preventing cross-workspace memory contamination, with configurable boundary rules.

**Key Files:**
- `src/workspace-boundary.ts` (4KB) - boundary enforcement
- `src/admission-control.ts` (21KB) - admission control

**Priority:** SECONDARY

---

### 11. Reflection System
**Description:** Learning from agent behavior patterns with reflection slices, event stores, and governance metadata for self-improvement.

**Key Files:**
- `src/reflection-store.ts` (21KB)
- `src/reflection-slices.ts` (12KB)
- `src/reflection-event-store.ts` (2KB)
- `src/intent-analyzer.ts` (8KB)

**Priority:** SECONDARY

---

### 12. Long-Context Chunking
**Description:** Intelligent chunking strategy for processing long documents with configurable overlap and size limits.

**Key Files:**
- `src/chunker.ts` (8KB)
- `docs/long-context-chunking.md`

**Priority:** SECONDARY

---

## Feature Priority Matrix

| Feature | Priority | Key Differentiator |
|---------|----------|-------------------|
| Hybrid Retrieval | CORE | Vector + BM25 fusion |
| Cross-Encoder Reranking | CORE | Multi-provider support |
| Multi-Scope Isolation | CORE | Agent/user/project boundaries |
| Smart Extraction | CORE | 6-category LLM classification |
| Lifecycle Management | CORE | Weibull decay + tier promotion |
| Auto-Capture/Recall | CORE | Transparent memory without prompting |
| Management CLI | SECONDARY | Production-ready tooling |
| Multi-Provider Embedding | SECONDARY | Any OpenAI-compatible API |
| Session Memory | SECONDARY | Cross-session continuity |
| Workspace Boundary | SECONDARY | Privacy isolation |
| Reflection System | SECONDARY | Self-improvement learning |
| Long-Context Chunking | SECONDARY | Large document handling |

---

## Architecture Summary

```
index.ts (Plugin Entry Point)
├── src/store.ts         (LanceDB Storage)
├── src/embedder.ts       (Embedding Abstraction)
├── src/retriever.ts      (Hybrid Retrieval Engine)
├── src/scopes.ts         (Multi-Scope Isolation)
├── src/tools.ts          (MCP Agent Tools)
├── src/smart-extractor.ts (LLM-Powered Extraction)
├── src/decay-engine.ts   (Weibull Decay)
├── src/tier-manager.ts   (Tier Promotion)
├── src/noise-filter.ts   (Content Filtering)
├── src/adaptive-retrieval.ts (Retrieval Decision)
├── src/session-compressor.ts (Session Memory)
├── src/reflection-*.ts   (Reflection System)
└── cli.ts                (Management CLI)
```

---

## Dependencies

| Package | Role |
|---------|------|
| `@lancedb/lancedb` | Vector database (ANN + FTS) |
| `openai` | OpenAI-compatible API client |
| `@sinclair/typebox` | JSON Schema type definitions |
| `apache-arrow` | Arrow data handling |
| `proper-lockfile` | Cross-process locking |

---

## CLI Commands

```bash
openclaw memory-pro list/search/stats/delete/delete-bulk/export/import/reembed/upgrade/migrate
openclaw memory-pro auth login/status/logout
```

---

*Generated from repository analysis - README.md, src/ directory, and package.json*
