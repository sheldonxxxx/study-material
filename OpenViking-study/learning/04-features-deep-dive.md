# OpenViking Feature Deep Dive

**Project:** OpenViking - Context Database for AI Agents
**Repository:** /Users/sheldon/Documents/claw/reference/OpenViking
**Generated:** 2026-03-27

---

## Overview

OpenViking is a context database designed for AI Agents, implementing a "filesystem paradigm" to unify memories, resources, and skills management. The system uses tiered context loading (L0/L1/L2), directory recursive retrieval, and automatic session management.

This document synthesizes all feature research into a coherent narrative ordered by architectural dependency. The virtual filesystem is the foundational layer; everything else builds upon it.

---

## Core Feature 1: Virtual Filesystem (viking:// Protocol)

**The foundational abstraction layer that unifies all context management under a filesystem paradigm.**

### What It Does

Every piece of information in OpenViking is identified by a `viking://` URI and accessed through standard filesystem-like operations (read, write, ls, mkdir, rm, mv). This provides a uniform interface for memories, resources, skills, and sessions.

### How It Works

**URI Format:** `viking://{scope}/{path}`

Valid scopes are: `resources`, `user`, `agent`, `session`, `queue`, `temp`

The `VikingFS` class (`openviking/storage/viking_fs.py`, 1910 lines) wraps an AGFS (Abstract Google File System) client and provides:
- URI-to-path mapping with account isolation
- L0/L1 reading (abstract, overview)
- Relation management
- Semantic search integration
- Vector store synchronization

**Account Isolation:** Each account's data is isolated under `/local/{account_id}/`. The `_ls_entries` method uses whitelisting at the account root level - only `VALID_SCOPES` are visible at the top level.

**Security Features:**
- URI traversal protection blocks `.`, `..`, `\`, and drive-prefixed components
- Access control enforced per scope: `resources` and `temp` are public, `_system` is forbidden, others use space-based isolation

**File Operations:**
- `read()` supports encryption with proper offset/size handling on decrypted content
- `write()` automatically creates parent directories
- `rm()` is idempotent (succeeds on non-existent paths after cleaning orphan index records)
- `mv()` implemented as copy + delete (avoids lock file carryover but doubles I/O)

### Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `openviking/storage/viking_fs.py` | 1910 | Main VikingFS implementation |
| `openviking_cli/utils/uri.py` | - | URI validation and parsing |
| `openviking/storage/vikingdb_manager.py` | - | Database manager |
| `openviking/storage/local_fs.py` | - | Local filesystem adapter |

### Notable Patterns

1. **Context variable binding:** Uses Python's `contextvars` for thread-safe request context propagation
2. **Batch abstract fetching:** Parallel fetch with semaphore limit of 6 concurrent operations
3. **Filename byte limit handling:** Components shortened if UTF-8 encoding exceeds 255 bytes using SHA256 hash suffix
4. **Graceful fallback for AGFS results:** Multiple checks for bytes vs objects with `.content` attribute

### Technical Debt

1. Synchronous AGFS calls in async methods - no async variants exist
2. Idempotent rm behavior is unusual but intentional
3. mv as cp+rm doubles I/O for large files
4. Large file (1910 lines) with many responsibilities

---

## Core Feature 2: Tiered Context Loading (L0/L1/L2)

**Three-tier information model balancing retrieval efficiency and content completeness.**

### What It Does

Automatically processes content into three hierarchy levels:
- **L0 (Abstract):** ~100 tokens, stored in `.abstract.md` - for vector search and quick filtering
- **L1 (Overview):** ~2KB, stored in `.overview.md` - for reranking and content navigation
- **L2 (Detail):** Full original content, unlimited - loaded on-demand

### How It Works

When content is added, the `SemanticProcessor` generates L0/L1 bottom-up: leaf nodes first, then parent directories aggregate child L0s into their own L1.

**Session Compression Integration:** The `SessionCompressor` (`openviking/session/compressor.py`) handles memory extraction with 6-category classification: PROFILE, PREFERENCES, ENTITIES, PATTERNS, TOOLS, SKILLS.

**Chunking Strategy:** Long memories are split into overlapping chunks:
- Each chunk gets `#chunk_NNNN` suffix in URI
- Parent pointer set to original memory URI
- Overlap ensures context continuity

**LLM-based Deduplication:** Before storing new memories, the `MemoryDeduplicator` checks for duplicates using LLM similarity analysis.

### Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `openviking/storage/compressor.py` | 30KB | V1 compression engine |
| `openviking/storage/compressor_v2.py` | - | V2 compression |
| `openviking/session/memory_deduplicator.py` | - | LLM-assisted deduplication |
| `openviking/session/compressor.py` | - | Session memory extraction |

### Notable Patterns

1. **Bottom-up processing:** L0/L1 generated from leaves upward
2. **Chunking with overlap:** For long content indexing
3. **6-category extraction:** Structured memory classification
4. **Pending semantic changes batched** and flushed after memory operations

### Technical Debt

1. Token estimation uses rough `len(msg.content) // 4` approximation
2. Large compressor files with multiple responsibilities
3. Synchronous wrappers for async methods add complexity

---

## Core Feature 3: Directory Recursive Retrieval

**Hierarchical retrieval combining vector search with directory traversal using a priority queue.**

### What It Does

The `HierarchicalRetriever` (`openviking/retrieve/hierarchical_retriever.py`, 619 lines) provides sophisticated semantic search that understands full context of information location. It uses score propagation from parent directories to children.

### How It Works

**Retrieval Flow:**
1. Determine starting directories by context_type
2. Global vector search to locate initial directories
3. Merge starting points + Rerank scoring
4. Recursive search using priority queue
5. Convert to MatchedContext

**Score Propagation Formula:**
```
final_score = alpha * embedding_score + (1 - alpha) * parent_score
```
Where `alpha = 0.5` (SCORE_PROPAGATION_ALPHA). Child scores inherit 50% from their parent directory's score, forming semantic clusters.

**Convergence Detection:** Algorithm stops when top-k URIs remain unchanged for 3 consecutive iterations AND we have at least `limit` candidates.

**Two Query Modes:**
- `find()`: Single query, low latency, simple use cases
- `search()`: LLM-based intent analysis generates 0-5 TypedQueries, higher latency but handles complex tasks

### Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `openviking/retrieve/hierarchical_retriever.py` | 619 | Main retriever |
| `openviking/retrieve/intent_analyzer.py` | - | Query intent analysis |
| `openviking/retrieve/retrieval_stats.py` | 8KB | Retrieval statistics |

### Notable Patterns

1. **Score propagation:** Semantic clusters via parent-child score inheritance
2. **Priority queue exploration:** Directories explored in score order
3. **Level-based termination:** L2 files are terminal nodes; only directories recurse
4. **Hotness boost:** Recently/actively accessed content ranked higher
5. **Sparse vector support:** Hybrid dense+sparse embedding search

### Technical Debt

1. `grep_patterns` implementation not clearly visible despite comments
2. Falls back to vector scores on rerank failure - may hide issues
3. Priority queue tie-breaking uses negative score (works but non-obvious)

---

## Core Feature 4: Vector Database Integration

**Pluggable backend system for semantic search across multiple vector database providers.**

### What It Does

Implements adapter pattern to abstract provider-specific details while exposing unified API for vector operations. Supports: Local, HTTP, Volcengine (hosted VikingDB), and VikingDB Private deployments.

### How It Works

**Architecture:**
```
VikingVectorIndexBackend (facade)
    └── CollectionAdapter (abstract base)
        ├── LocalCollectionAdapter
        ├── HttpCollectionAdapter
        ├── VolcengineCollectionAdapter
        └── VikingDBPrivateCollectionAdapter
```

**URI Encoding/Decoding:** The adapter transforms `viking://user/space/path` to `/user/space/path` for backend storage while preserving full URIs for display.

**Tenant Isolation:** The facade enforces account-based data isolation by automatically injecting account filters on every query.

**RocksDB LOCK Contention Avoidance:** Shares a single adapter (and its underlying PersistStore/RocksDB instance) across all account backends.

### Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `openviking/storage/viking_vector_index_backend.py` | 1014 | Main facade |
| `openviking/storage/vectordb_adapters/base.py` | 520 | Abstract base |
| `openviking/storage/vectordb_adapters/factory.py` | - | Registry factory |

### Notable Patterns

1. **Filter Expression Tree:** Type-safe filter composition with And/Or/Not logic
2. **Adapter Registry:** Dynamic class loading with fallback for custom backends
3. **Shared Adapter Pattern:** Prevents file locking contention
4. **Automatic Account Injection:** Tenant isolation without requiring callers to add filters

### Technical Debt

1. Filter compilation logic duplicated in adapter base class
2. Sync/async mixture with inconsistent patterns
3. Broad exception catching may mask underlying issues

---

## Core Feature 5: Session Management & Memory Self-Iteration

**Automatic conversation compression, memory extraction, and iterative context improvement.**

### What It Does

Transforms raw conversations into structured long-term memories organized by category. Sessions automatically archive, extract memories via LLM, and iteratively improve context.

### How It Works

**Session URI Structure:**
```
viking://session/{user_space}/{session_id}/
├── messages.jsonl          # Current messages
├── .meta.json              # Session metadata
├── .abstract.md            # L0 summary
├── .overview.md            # L1 overview
├── history/
│   └── archive_001/        # Archived messages
└── tools/
    └── {tool_id}/          # Tool statistics
```

**Two-Phase Commit:**
1. **Phase 1 (Lock-protected):** Copy messages, clear live session, increment compression index
2. **Phase 2 (Background):** Memory extraction runs via `asyncio.create_task()`

**Memory Categories:**
- UserMemory: PROFILE, PREFERENCES, ENTITIES, EVENTS
- AgentMemory: CASES (problems + solutions), PATTERNS (reusable processes)
- Tool/Skill Memory: TOOLS, SKILLS

**Language Detection:** Progressive detection scoped to user messages only to avoid assistant text bias. Handles CJK disambiguation, Korean, Cyrillic, Arabic, and more.

### Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `openviking/session/session.py` | 1025 | Main session class |
| `openviking/session/memory_extractor.py` | 900+ | LLM-based extraction |
| `openviking/session/memory_deduplicator.py` | - | LLM-assisted deduplication |
| `openviking/session/compressor.py` | - | Session compression |

### Notable Patterns

1. **Distributed Lock with Pre-check:** Avoids lock acquisition for common empty-message case
2. **Background Memory Extraction:** Phase 2 runs async so commit returns immediately
3. **Profile Append Pattern:** Profile uses append-merge rather than create-new
4. **Monotonic Statistics Tracking:** Tool memory merge validates that total_calls never decreases

### Technical Debt

1. Large memory extractor (900+ lines) with multiple responsibilities
2. Token estimation uses `len(msg.content) // 4`
3. Profile merge edge cases could produce unexpected results

---

## Core Feature 6: VikingBot AI Agent Framework

**Multi-channel AI agent with tool execution, memory integration, and session management.**

### What It Does

Built on OpenViking, VikingBot provides channels (CLI, Telegram, Slack, Discord, etc.), agent implementations, and OpenViking mount integration for context access.

### How It Works

**Architecture:**
```
AgentLoop (main processing engine)
    ├── ContextBuilder (system prompt from bootstrap files, memory, skills)
    ├── MemoryStore (two-layer: MEMORY.md + HISTORY.md)
    ├── SessionManager (per-session OpenViking mounts)
    ├── ToolRegistry (tool definitions and execution)
    └── SubagentManager (subagent orchestration)

Channels (BaseChannel abstract)
    ├── TelegramChannel
    ├── SlackChannel
    ├── DiscordChannel
    └── ... (many more)

LLM Providers (LLMProvider abstract)
    ├── LiteLLMProvider (LiteLLM library routing)
    └── OpenAICompatibleProvider (Direct OpenAI SDK)
```

**Message Processing Pipeline:**
1. Handle slash commands (/new, /remember, /help)
2. Consolidate if session too large (clone + archive)
3. Build context from bootstrap files, memory, skills
4. Run agent loop with parallel tool execution
5. Save session

**Parallel Tool Execution:**
```python
tool_tasks = [execute_single_tool(idx, tool_call) for idx, tool_call in enumerate(response.tool_calls)]
results = await asyncio.gather(*tool_tasks)
```

**Progressive Skills Loading:** Always-loaded skills included fully; available skills show summary only.

### Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `bot/vikingbot/agent/loop.py` | 762 | Main agent loop |
| `bot/vikingbot/agent/context.py` | 356 | Context builder |
| `bot/vikingbot/agent/memory.py` | - | Memory store |
| `bot/vikingbot/channels/base.py` | - | Abstract channel |
| `bot/vikingbot/providers/base.py` | - | Abstract provider |

### Notable Patterns

1. **Event Bus:** Async message bus for loose coupling
2. **Session Cloning:** For async operations without blocking
3. **Processing Ticks:** Periodic "still working" events for long operations
4. **Sandbox Isolation:** Tools execute in restricted filesystem environment
5. **Group Chat Detection:** Handles `@user` mentions differently

### Technical Debt

1. Channel implementations vary significantly
2. ToolRegistry appears to be global singleton
3. Large agent loop (762 lines) handles many concerns
4. Magic string commands hardcoded rather than enum

---

## Core Feature 7: Multi-Provider Model Support

**Unified support for VLM and embedding models across 15+ providers.**

### What It Does

Provides a layered architecture for multi-provider support: Registry + LiteLLM Provider + OpenAI-Compatible Provider. Supports gateways (OpenRouter, AihubMix) and local deployments (vLLM, Ollama).

### How It Works

**Provider Detection Logic:**
1. Direct match by config key
2. Auto-detect by api_key prefix (e.g., `sk-or-` for OpenRouter)
3. Auto-detect by api_base URL keyword

**Model Resolution:**
```python
def _resolve_model(self, model: str) -> str:
    if self._gateway:
        # Gateway mode: strip provider prefix, re-prefix with gateway
        prefix = self._gateway.litellm_prefix
        if self._gateway.strip_model_prefix:
            model = model.split("/")[-1]
        if prefix and not model.startswith(f"{prefix}/"):
            model = f"{prefix}/{model}"
        return model
    # Standard mode: auto-prefix for known providers
```

**System Message Handling:** MiniMax doesn't support system messages - they merge into the first user message.

**Model-Specific Overrides:** Kimi K2.5 requires `temperature >= 1.0` (hardcoded).

### Supported Providers

| Provider | Prefix | Notes |
|----------|--------|-------|
| OpenRouter | openrouter | Gateway, detects by `sk-or-` key |
| AihubMix | openai | Gateway, strips model prefix |
| Anthropic | (none) | Native LiteLLM |
| OpenAI | (none) | Native LiteLLM |
| DeepSeek | deepseek | |
| VolcEngine | volcengine | |
| Gemini | gemini | |
| Zhipu | zai | Mirrors key to ZHIPUAI_API_KEY |
| DashScope | dashscope | |
| Moonshot | moonshot | K2.5 requires temp >= 1.0 |
| MiniMax | minimax | System messages merged |
| vLLM | hosted_vllm | Local deployment |
| Groq | groq | |

### Key Files

| File | Purpose |
|------|---------|
| `bot/vikingbot/providers/registry.py` | ProviderSpec definitions |
| `bot/vikingbot/providers/litellm_provider.py` | LiteLLM routing |
| `bot/vikingbot/providers/openai_compatible_provider.py` | Direct SDK |

### Notable Patterns

1. **Gateway Priority:** Gateways matched first since they can route any model
2. **Skip Prefixes:** Prevents double-prefixing when models already include provider prefix
3. **Environment Variable Mirroring:** ZhipuAI key mirrored to satisfy different LiteLLM paths
4. **API Key Prefix Detection:** OpenRouter detected by `sk-or-` without model keywords

### Technical Debt

1. LiteLLM + Direct SDK Hybrid - potential for consolidation
2. Multiple detection strategies may cause edge cases
3. Hardcoded model overrides (K2.5 temperature)

---

## Secondary Feature 8: Rust CLI (ov_cli)

**High-performance Rust-based command-line client with TUI for interactive use.**

### What It Does

Alternative to Python client with TUI for interactive file browsing, rich output formatting (table, JSON, simple), and 18 command categories.

### How It Works

**Architecture:**
```
crates/ov_cli/src/
├── main.rs          # Entry point, command routing (35KB)
├── client.rs        # HTTP client with all API methods (26KB)
├── commands/        # Individual command handlers
│   ├── filesystem.rs # ls/tree/mkdir/rm/mv/stat
│   ├── search.rs    # find/search/grep/glob
│   ├── session.rs   # Session management
│   └── ... (14 more)
└── tui/             # Terminal UI (ratatui + crossterm)
```

**TUI Implementation:** Uses `crossterm` for terminal control and `ratatui` for rendering with panic hook to ensure terminal restore on crash.

**Directory Zipping:** Local directories zipped before upload using `zip` crate with `WalkDir` traversal.

### Key Commands

| Command | Description |
|---------|-------------|
| `ov add-resource` | Add local file/URL as resource |
| `ov ls/tree` | List directory contents |
| `ov read/abstract/overview` | Read content at L0/L1/L2 |
| `ov find/search` | Semantic search |
| `ov session new/list/get/delete` | Session management |
| `ov tui` | Interactive TUI file explorer |
| `ov chat` | Chat with vikingbot agent |

### Notable Patterns

1. **Panic Hook for Terminal Restore:** Ensures terminal state even on crash
2. **Path Unescaping:** Handles shell-escaped spaces with helpful suggestions
3. **Machine ID for Session:** Uses `machine_uid` crate for persistent session IDs
4. **Configurable Output Formats:** Table, JSON, simple with compact mode

### Technical Debt

1. Large main.rs (35KB) with centralized command routing
2. No connection pooling visibility
3. Synchronous ZIP in async context

---

## Secondary Feature 9: HTTP Server & REST API

**FastAPI-based HTTP server providing persistent context API for AI agents.**

### What It Does

Multi-tenant server with API key authentication, 16 router categories, Bot API proxy to Vikingbot gateway, and Prometheus telemetry integration.

### How It Works

**Router Architecture:**
```
/api/v1/admin      # Account/user management (ROOT/ADMIN)
/api/v1/resources # Add resources, temp upload
/api/v1/fs        # ls/tree/mkdir/rm/mv/stat
/api/v1/search    # find/search/grep/glob
/api/v1/sessions  # Session CRUD
/api/v1/bot       # Bot API proxy
```

**Auth Modes:**
1. `api_key` with manager (production): API keys resolved via `APIKeyManager`
2. `api_key` without manager (dev): Implicit ROOT/default identity
3. `trusted` (gateway): Trust `X-OpenViking-Account/User` headers

**Three Auth Modes:**
- **api_key with manager:** Production mode with APIKeyManager
- **api_key without manager:** Dev mode with implicit ROOT
- **trusted:** Gateway mode trusting headers (logs warning if non-localhost)

### Key Files

| File | Purpose |
|------|---------|
| `openviking/server/app.py` | FastAPI application factory |
| `openviking/server/bootstrap.py` | Server bootstrap |
| `openviking/server/routers/` | 16 router categories |
| `openviking/server/config.py` | Server configuration |

### Notable Patterns

1. **Factory Mode for Multi-Worker:** Uses import string for worker processes
2. **Bot Process Management:** Spawns vikingbot as subprocess with log handling
3. **Dependency Injection:** Global `_service` via `set_service()` / `get_service()`
4. **Telemetry Wrapper:** `run_operation()` wraps all operations for metrics
5. **Float Sanitization:** Replaces inf/nan with 0.0 for JSON compliance

### Technical Debt

1. ROOT API Key in Dev Mode - server refuses to start if not configured (correct behavior)
2. Graceful shutdown - bot subprocess termination could timeout

---

## Secondary Feature 10: AGFS (Abstract Google File System) C++ Backend

**Plugin-based filesystem abstraction with high-performance C++ server.**

### What It Does

AGFS provides a virtual filesystem layer with plugin architecture allowing different backends (local, S3, memory) to be mounted. The C++ server (`agfs-server`) provides high-performance file operations.

### How It Works

**Plugin Architecture:**
```
/serverinfo  # Server metadata (serverinfofs)
/queue       # Async queue with SQLite backend (queuefs)
/local       # Storage (localfs/s3fs/memfs)
```

**HandleFS:** POSIX-like file handles with lease management for stateful access patterns.

**Python SDK:** `pyagfs/client.py` (35KB) provides pure HTTP client with optional native binding fallback.

**Configuration:**
```python
class AGFSConfig:
    port: int = 1833
    backend: str = "local"  # 'local' | 's3' | 'memory'
    mode: str = "binding-client"  # 'http-client' | 'binding-client'
```

### Notable Patterns

1. **Plugin Architecture:** Different backends mounted at different paths
2. **HandleFS with Leases:** POSIX-like operations with renewal
3. **Optional Native Binding:** Gracefully degrades to HTTP client
4. **Retry with Exponential Backoff:** 1s, 2s, 4s for writes

### Technical Debt (Documented in Code)

1. **SQLite queue backend is single-node only:** Each AGFS instance gets isolated queue.db - no cross-instance messaging
2. **TiDB backend lacks at-least-once delivery:** Ack() and RecoverStale() are no-ops
3. **Multi-node not supported:** For production multi-node, switch backend to TiDB or MySQL

---

## Secondary Feature 11: OpenClaw/OpenCode Plugin Integration

**Context plugins demonstrating OpenViking as drop-in memory backend for external agent frameworks.**

### What It Does

Two plugin implementations:
- **OpenClaw Plugin 2.0:** Context-engine architecture with auto-recall/capture
- **OpenCode Memory Plugin:** Tool-based memory access (memsearch, memread, membrowse, memcommit)

### How It Works

**OpenClaw Plugin:**
- Registers as `context-engine` kind
- Spawns local OpenViking subprocess OR connects to remote
- Hooks: `session_start`, `session_end`, `before_prompt_build`, `before_reset`
- Auto-recall injects memories before prompt building

**OpenCode Plugin:**
- Four explicit tools: `memsearch`, `memread`, `membrowse`, `memcommit`
- Session mapping with persistence to `openviking-session-map.json`
- Token budget enforcement for memory lines

### Comparison

| Aspect | OpenClaw Plugin 2.0 | OpenCode Memory Plugin |
|--------|---------------------|----------------------|
| Architecture | Context Engine | Tool-based |
| Memory Access | Auto-inject via hooks | Explicit tool calls |
| Session Management | Context engine factory | Session mapping with persistence |
| Auto-recall | `before_prompt_build` hook | Tool-based (agent decides) |
| Auto-capture | `afterTurn` hook | `memcommit` tool |
| Deployment | Local (subprocess) or Remote | Remote only |

### Technical Debt

1. OpenClaw Plugin 2.0 not backward-compatible with legacy plugin
2. Historical compatibility issue with OpenClaw `2026.3.12` causing conversation hangs
3. Auto-recall has 5-second timeout protection to prevent agent hangs

---

## Secondary Feature 12: Observability & Telemetry

**Operation-scoped telemetry collection, retrieval statistics, and evaluation capabilities.**

### What It Does

Low-overhead telemetry system with operation collectors, retrieval statistics, Prometheus integration, and RAGAS-based evaluation framework.

### How It Works

**OperationTelemetry:** Disabled-by-default collector with zero overhead when disabled. Uses counters (monotonic), gauges (latest), and duration tracking.

**ContextVar Binding:** Request-scoped telemetry without explicit passing through call chains:
```python
_CURRENT_TELEMETRY: ContextVar[OperationTelemetry] = ContextVar(
    "openviking_operation_telemetry",
    default=_NOOP_TELEMETRY,
)
```

**RetrievalStatsCollector:** Thread-safe singleton for retrieval metrics with Prometheus observer integration.

### Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `openviking/telemetry/operation.py` | 15KB | Core collector |
| `openviking/retrieve/retrieval_stats.py` | 8KB | Retrieval metrics |
| `openviking/eval/` | - | RAGAS evaluation |

### Notable Patterns

1. **Disabled-by-Default:** Zero overhead in production
2. **Thread-Safe Collectors:** All use locks for concurrent access
3. **ContextVar Binding:** Request-scoped without explicit passing
4. **Prometheus Integration:** Optional observer for metrics export
5. **Zero-Copy Snapshots:** Uses `copy.deepcopy` for stats to avoid lock contention

### Technical Debt

1. Event capture intentionally disabled for summary-only telemetry
2. Prometheus observer import in try/except to avoid hard dependency
3. Token estimation uses chars/4 heuristic

---

## Cross-Feature Architecture

### Dependency Graph

```
Virtual Filesystem (viking://)
    |
    +-- Tiered Context Loading (L0/L1/L2)
    |       |
    |       +-- Session Management (creates L0/L1)
    |
    +-- Directory Recursive Retrieval
    |       |
    |       +-- Vector DB (semantic search backend)
    |
    +-- Session Management
    |       |
    |       +-- AGFS Backend (memory storage)
    |
    +-- HTTP Server (exposes VikingFS operations)
    |
    +-- VikingBot (mounts OpenViking context)
            |
            +-- Multi-Provider (LLM routing)
```

### Data Flow: Adding a Resource

1. Resource content arrives via API or CLI
2. Parser extracts structured content
3. SemanticProcessor generates L0/L1 bottom-up
4. Content stored via VikingFS `write()`:
   - L2: `{uri}/content.md`
   - L0: `{uri}/.abstract.md`
   - L1: `{uri}/.overview.md`
5. Vector embeddings created from L0 text
6. VectorDB updated with URI, level, parent_uri

### Data Flow: Retrieval

1. User issues query via `find()` or `search()`
2. If `search()`: IntentAnalyzer generates TypedQueries from session context
3. HierarchicalRetriever:
   - Global vector search finds initial candidates
   - Priority queue explores directories recursively
   - Scores propagate from parent to children
   - Convergence detected after stable top-k
4. Results include URI, level, abstract, relations
5. Client reads L2 content on-demand via VikingFS

---

## Key Implementation Patterns

| Pattern | Where Used | Purpose |
|---------|------------|---------|
| Adapter Pattern | Vector DB | Backend abstraction |
| Two-Phase Operations | Session commit | Lock-protected sync + async background |
| Expression Trees | Filters, Memory | Type-safe composition |
| Event Bus | VikingBot | Loose coupling |
| Context Builders | VikingBot | Assembling prompts from multiple sources |
| URI Encoding | VikingFS | Storage format abstraction |
| Priority Queue | HierarchicalRetriever | Score-ordered exploration |
| ContextVar Binding | Telemetry, VikingFS | Request-scoped without explicit passing |
| Handle with Leases | AGFS | Stateful file access |
| Registry Factory | Providers, Vector DB | Dynamic backend selection |

---

## Summary

OpenViking implements a sophisticated context management system with the virtual filesystem as its foundational layer. Key architectural insights:

1. **Filesystem Paradigm:** Unified URI-based access to all context types
2. **Tiered Loading:** L0/L1/L2 balances efficiency and completeness
3. **Hierarchical Retrieval:** Score propagation creates semantic clusters
4. **Automatic Memory:** Sessions self-archive and extract long-term memories
5. **Multi-Agent Support:** VikingBot provides production-ready agent framework
6. **Provider Abstraction:** 15+ providers via LiteLLM and custom routing
7. **Observability:** Low-overhead telemetry for production debugging

The system demonstrates mature software architecture with clear separation of concerns, comprehensive error handling, and thoughtful technical debt management.
