# Project Overview: memory-lancedb-pro

## What the Project Is

**memory-lancedb-pro** is an enhanced long-term memory plugin for Claude Code (OpenClaw), built on LanceDB for vector storage with hybrid retrieval capabilities. It provides persistent, searchable memory across Claude Code sessions with advanced features like multi-scope isolation, intelligent fact extraction, and memory lifecycle management.

## Core Value Proposition

The plugin solves a fundamental problem: Claude Code sessions are inherently ephemeral, losing all context when they end. memory-lancedb-pro provides:

- **Persistent Memory**: Memories survive session boundaries and can be recalled in future sessions
- **Hybrid Search**: Combines vector similarity search with BM25 full-text search for accurate retrieval
- **Intelligent Extraction**: Automatically extracts facts, preferences, and patterns from conversations
- **Multi-Scope Isolation**: Memory can be isolated per-agent, per-project, per-user, or shared globally
- **Cross-Encoder Reranking**: Uses machine learning reranking to improve search result quality

## Target Users

1. **Individual Claude Code Users**: Want Claude to remember preferences, project context, and accumulated knowledge across sessions
2. **Development Teams**: Share memory across team members working on the same codebase
3. **Power Users**: Use custom scopes to organize memory by project, client, or any logical boundary

## Key Differentiators from Basic Memory Systems

| Feature | Basic Memory | memory-lancedb-pro |
|---------|--------------|-------------------|
| **Storage** | Simple key-value or JSON | LanceDB with vector indexing |
| **Search** | Keyword only | Hybrid vector + BM25 + reranking |
| **Isolation** | Global only | 7 scope types (global, agent, project, user, custom, team, reflection) |
| **Extraction** | Manual storage | Automatic smart extraction with intent analysis |
| **Lifecycle** | Unlimited growth | Tier management, compaction, decay engine |
| **Access Control** | None | Scope-based access validation |
| **Recovery** | None | Session recovery, reflection bypass hooks |
| **CLI** | None | Full management CLI with OAuth login |

## Architecture Overview

```
User Interaction
      │
      ▼
OpenClaw Plugin (index.ts)
      │
      ├──► embedder.ts ──► OpenAI-compatible API
      │
      ├──► store.ts ──► LanceDB (local filesystem)
      │
      ├──► retriever.ts ──► Hybrid search + reranking
      │
      ├──► smart-extractor.ts ──► Fact extraction
      │
      └──► tools.ts ──► MCP tools for Claude Code
```

## Technical Stack

| Component | Technology |
|-----------|------------|
| Runtime | Node.js ESM (TypeScript) |
| Database | LanceDB (embedded, local) |
| Embeddings | OpenAI-compatible API (Azure OpenAI, Ollama, etc.) |
| Authentication | OAuth 2.0 with PKCE |
| Validation | TypeBox (JSON Schema runtime validation) |
| CLI | Commander.js |

## Version & Status

- **Current Version**: 1.1.0-beta.10
- **License**: MIT
- **Maturity**: Production beta with active development

## File Structure Summary

| Path | Purpose |
|------|---------|
| `index.ts` | Main plugin entry (155KB) |
| `cli.ts` | CLI management tool (47KB) |
| `openclaw.plugin.json` | Plugin manifest |
| `src/` | 45 TypeScript modules (~16K LOC) |
| `test/` | 49 test files |

## Key Modules

- **store.ts**: LanceDB persistence layer with cross-process locking
- **embedder.ts**: Embedding generation with LRU cache and multi-key rotation
- **retriever.ts**: Hybrid retrieval with RRF fusion and cross-encoder reranking
- **scopes.ts**: Multi-scope isolation and access control
- **smart-extractor.ts**: Intelligent fact extraction from conversations
- **llm-oauth.ts**: OAuth 2.0 PKCE authentication for LLM providers
- **memory-compactor.ts**: Memory cleanup and tier management
- **decay-engine.ts**: Memory importance decay calculations

## Supported Use Cases

1. **Project Context Recall**: Remember project architecture, decisions, and conventions
2. **Preference Memory**: Recall user preferences for code style, tooling, workflows
3. **Cross-Session Learning**: Build on previous session outcomes
4. **Team Knowledge Sharing**: Share patterns and learnings across team members
5. **Reflection Storage**: Store agent self-reflection for meta-cognition
