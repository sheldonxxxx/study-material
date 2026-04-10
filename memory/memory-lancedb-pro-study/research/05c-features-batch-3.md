# Feature Batch 3 Deep Dive (Features 7-9)

## Feature 7: Management CLI

### Description
Full CLI toolkit via `openclaw memory-pro` providing list, search, stats, delete, delete-bulk, export, import, reembed, upgrade, migrate, auth, and reindex-fts commands.

### Key Implementation Files

**`cli.ts` (1,350 lines)**
- Main CLI entry point
- Uses `commander` package for command registration
- Context-driven architecture: `CLIContext` injected with store, retriever, scopeManager, migrator, embedder, llmClient
- Factory function: `createMemoryCLI(context)` returns registration function

**`src/tools.ts` (74KB)**
- MCP tools definition for agent integration
- Not reviewed in detail due to size

### Core Command Implementations

| Command | Lines | Description |
|---------|-------|-------------|
| `memory-pro list` | 649-691 | List memories with scope/category filtering, pagination |
| `memory-pro search <query>` | 693-736 | Hybrid retrieval search with result scoring |
| `memory-pro stats` | 738-789 | Aggregate statistics across scopes and categories |
| `memory-pro delete <id>` | 791-815 | Single memory deletion with scope access control |
| `memory-pro delete-bulk` | 817-856 | Bulk delete with --dry-run safety, date filtering |
| `memory-pro export` | 858-905 | JSON export excluding vectors for reduced size |
| `memory-pro import <file>` | 907-1037 | Import with idempotency (id-based + 0.95 similarity dedupe) |
| `memory-pro reembed` | 1039-1162 | Re-embed from source LanceDB to target (A/B testing) |
| `memory-pro upgrade` | 1164-1229 | Upgrade legacy memories to 6-category smart format |
| `memory-pro migrate check/run/verify` | 1231-1319 | Legacy migration workflow |
| `memory-pro auth login/status/logout` | 468-647 | OAuth authentication management |
| `memory-pro reindex-fts` | 1321-1340 | Rebuild BM25 full-text search index |
| `memory-pro version` | 460-466 | Print plugin version |

### Notable Patterns

**OAuth Authentication Flow (lines 472-647)**
- OAuth providers via `llm-oauth.ts` module
- Interactive TUI for provider selection (arrow keys + Enter)
- Backs up existing LLM config before switching to OAuth mode
- On logout, restores from backup or builds fallback config
- Stores OAuth token in `~/.openclaw/.memory-lancedb-pro/oauth.json`
- Config update path: `~/.openclaw/openclaw.json` via JSON5 parsing

**Import Idempotency (lines 982-999)**
```
1. If import file has id and target DB already has it -> skip
2. If no id provided -> similarity check (score > 0.95) against existing -> skip
```

**Re-embed Safety (lines 1065-1078)**
- Prevents accidental in-place re-embedding
- Uses `fs.realpath` to resolve symlinks before comparison
- Requires `--force` flag to override

**CLI Search Retry (lines 442-453)**
- Sleeps 75ms then retries on empty results
- Handles timing issues with async embedding

### Technical Debt / Concerns

1. **OAuth logout fallback is imprecise** (lines 626-637): If no backup exists, `buildLogoutFallbackLlmConfig` extracts baseURL from OAuth config, which may not represent the original API-key endpoint

2. **Import similarity check uses 0.95 threshold** (line 995): Very strict threshold may miss semantically similar memories that should be deduplicated

3. **Re-embed loads entire source DB into memory** (line 1090): `toArray()` on potentially large tables without streaming

4. **No progress indicator for import** (lines 932-1032): Long imports show no feedback until completion

---

## Feature 8: Multi-Provider Embedding

### Description
Compatible with any OpenAI-compatible API provider including Jina, OpenAI, Voyage, Google Gemini, and Ollama (local).

### Key Implementation Files

**`src/embedder.ts` (942 lines)**
- `EmbeddingCache`: LRU cache with TTL (256 entries, 30 min)
- `EmbeddingCapabilities`: Provider capability mapping
- `Embedder`: Main class with multi-key rotation, retry logic, chunking

### Provider Detection Logic (lines 234-248)

```typescript
function detectEmbeddingProviderProfile(baseURL, model):
  - /api\.openai\.com/ -> "openai"
  - /\.openai\.azure\.com/ -> "azure-openai"
  - /api\.jina\.ai/ or /^jina-/ -> "jina"
  - /api\.voyageai\.com/ or /^voyage/ -> "voyage-compatible"
  - otherwise -> "generic-openai-compatible"
```

### Capability Mapping (lines 250-288)

| Provider | encoding_format | normalized | taskField | dimensionsField |
|----------|----------------|------------|----------|----------------|
| openai | true (float) | false | null | "dimensions" |
| jina | true | true | "task" | "dimensions" |
| voyage-compatible | false | false | "input_type" (with value map) | "output_dimension" |
| generic | true | false | null | "dimensions" |

### Key Algorithms

**Multi-Key Round-Robin Rotation (lines 483-488)**
```typescript
private nextClient(): OpenAI {
  const client = this.clients[this._clientIndex % this.clients.length];
  this._clientIndex = (this._clientIndex + 1) % this.clients.length;
  return client;
}
```

**Rate Limit Handling with Key Rotation (lines 519-555)**
- Tries each key at most once before giving up
- On 429/503 or `rate_limit_exceeded`/`insufficient_quota` codes -> rotate
- Other errors -> propagate immediately without retry

**Auto-Chunking for Long Documents (lines 659-776)**
- Triggered on context length errors
- Uses `smartChunk()` from `chunker.ts`
- Recursion depth limit: `MAX_EMBED_DEPTH = 3`
- Single chunk output detection: if chunk ~90% of original, force-truncate to 50%
- Averages embeddings across chunks

**Embedding Cache (lines 24-80)**
- SHA256 hash of `${task}:${text}`, truncated to 24 chars
- LRU eviction: oldest entry removed when full
- TTL: 30 minutes
- Hit rate tracking for diagnostics

### Task-Aware API (lines 592-610)

```typescript
embedQuery(text)      // Uses _taskQuery hint (e.g. "retrieval.query")
embedPassage(text)    // Uses _taskPassage hint (e.g. "retrieval.passage")
embedBatchQuery()     // Parallel batch with query task
embedBatchPassage()  // Parallel batch with passage task
```

### Error Handling Patterns (lines 311-361)

**Auth Errors**: 401/403 status, `invalid.*key|auth|forbidden|unauthorized` codes
**Network Errors**: `ECONNREFUSED|ECONNRESET|ETIMEDOUT|fetch failed`
**Provider-Specific Hints**:
- Jina: Suggest Ollama fallback if key expired
- Ollama: Verify local server running, model pulled, dimensions match

### Notable Technical Decisions

1. **Passage vs Query distinction**: Jina and Voyage support task-type hints that improve retrieval quality. The plugin maps `embedQuery` to `retrieval.query` and `embedPassage` to `retrieval.passage`.

2. **Dimensions field is provider-specific**: OpenAI uses `dimensions`, Voyage uses `output_dimension`, some local models reject it entirely (`omitDimensions` flag).

3. **Batch timeout not applied**: Individual batch texts are not wrapped with `withTimeout` because the overall API call is already bounded by the SDK.

4. **Force truncation on MAX_EMBED_DEPTH**: After 3 retries, text is deterministically halved to guarantee progress.

### Technical Debt / Concerns

1. **EMBEDDING_DIMENSIONS table is manually maintained** (lines 136-160): Adding new models requires code changes. No dynamic discovery.

2. **Cache key ignores API key**: Same text+task on different keys produces same cache hit, potentially returning wrong model embeddings if keys target different providers.

3. **No circuit breaker for failing providers**: After exhausting rate limits, all keys fail. No cooldown period or temporary bypass.

---

## Feature 9: Session Memory

### Description
Session summarization on `/new` command with configurable message count. Stores session summaries to LanceDB for cross-session continuity.

### Key Implementation Files

**`src/session-compressor.ts` (332 lines)**
- Scoring algorithm for text segments
- `compressTexts()` - greedy compression within character budget
- `estimateConversationValue()` - adaptive throttle decision

**`src/session-recovery.ts` (138 lines)**
- Session directory discovery
- `resolveReflectionSessionSearchDirs()` - finds all possible session locations

**`src/index.ts`** (usage integration)
- Line 52: imports `compressTexts`, `estimateConversationValue`
- Line 2678: logs session compression stats
- Hooks: `command:new`, `command:reset` for session boundaries

### Scoring Algorithm (session-compressor.ts, lines 102-150)

| Score | Reason | Pattern |
|-------|--------|---------|
| 1.0 | tool_call | `tool_use`, `tool_result`, `function_call`, memory operations |
| 0.95 | correction | `actually`, `instead`, `wrong`, CJK variants |
| 0.85 | decision | `let's go with`, `confirmed`, `approved`, `decided` |
| 0.3 | system_xml | XML tags with content (filtered as boilerplate) |
| 0.7 | substantive | >80 chars (or >30 for CJK), not system XML |
| 0.5 | short_question | Contains `?` or `\uff1f` |
| 0.4 | short_statement | Short non-question text |
| 0.1 | acknowledgment | `ok`, `okay`, `thanks`, `好的`, etc. |
| 0.0 | empty | Whitespace only |

### Compression Algorithm (session-compressor.ts, lines 171-277)

```
1. If all texts fit in budget -> return all
2. Always keep first and last text (session boundaries)
3. Score remaining texts, sort by score desc, then index asc
4. Identify paired tool_call + tool_result (indices i, i+1)
5. Greedily select highest-scoring until budget exhausted
6. When selecting a tool_call, also try to add its partner
7. If all scores < threshold, ensure at least minTexts (3)
8. Re-sort selected by original index (chronological order)
```

### Conversation Value Estimation (lines 289-331)

Used by adaptive extraction throttle to skip low-value conversations:

| Signal | Weight | Pattern |
|--------|--------|---------|
| Memory intent | +0.5 | `remember`, `don't forget`, `记住` |
| Tool calls | +0.4 | tool_use, tool_result patterns |
| Corrections/decisions | +0.3 | correction or decision indicators |
| Substantive content >200 chars | +0.2 | sum of texts >20 chars |
| Multi-turn (>6 texts) | +0.1 | texts.length > 6 |
| **Maximum** | 1.0 | Cap |

### Session Recovery (session-recovery.ts, lines 51-137)

Discovers session directories from multiple sources:

1. **From explicit paths**:
   - `currentSessionFile` directory
   - `context.previousSessionEntry.sessionFile/sessionsDir/sessionDir`
   - `context.sessionEntry.sessionFile/sessionsDir/sessionDir`
   - `workspaceDir/sessions`

2. **From environment/config**:
   - `OPENCLAW_HOME` env var
   - Derived from workspace path (`.../workspace/...` -> parent)
   - Derived from session file path (`.../agents/<id>/sessions/...` -> parent)

3. **Agent ID discovery**:
   - `sourceAgentId` parameter
   - `context.agentId`
   - `sessionEntry.agentId`
   - Configured agent IDs from openclaw.json
   - Default: `"main"`

4. **Cross-product**: For each `(home, agentId)` pair, adds `home/agents/<agentId>/sessions`

### Session Summary Storage

Session summaries are stored with:
- `category`: derived from the extraction (likely "case" or "event")
- `metadata.type`: `"session-summary"` (filtered in lifecycle, line 1799 of index.ts)
- `scope`: Current agent/project scope
- L0/L1/L2 tier based on lifecycle promotion

### Session Integration Flow

```
agent_end hook (auto-capture)
  -> extract messages from event
  -> estimateConversationValue() -> skip if < threshold
  -> compressTexts(maxChars) -> prioritize high-signal
  -> smart-extractor processes compressed text
  -> store to LanceDB

command:new hook
  -> resolveReflectionSessionSearchDirs()
  -> loadAgentReflectionSlicesFromEntries()
  -> runMemoryReflection() -> generate reflection log
```

### Technical Debt / Concerns

1. **CJK character threshold is approximate** (line 134): Uses 30 chars for CJK vs 80 for Latin, based on observation that CJK carries more meaning per character. Not validated against actual compression quality.

2. **Paired tool_call detection is fragile** (lines 229-240): Only looks at adjacent indices `i, i+1`. If session has interleaved tool calls or the result is not immediately after, pairing fails.

3. **Session recovery may find stale directories**: No validation that discovered session directories actually contain session files. Could lead to empty search results.

4. **estimateConversationValue thresholds are magic numbers**: 0.5 threshold for memory intent, 200 chars for substantive, 6 for multi-turn. Not configurable.

5. **Session summaries lose temporal ordering**: After compression and extraction, the summary is stored as a single memory entry without explicit ordering relative to other session summaries.

---

## Cross-Cutting Observations

### Shared Patterns

1. **Error propagation with context**: Both embedder and CLI wrap errors with `formatEmbeddingProviderError()` or similar, adding hints for common failure modes.

2. **Provider profile detection via regex**: Used in both embedder (baseURL/model matching) and likely elsewhere for provider-specific behavior.

3. **Configuration resolution chain**: CLI uses `explicit -> env var -> default` for paths, similar to other parts of the codebase.

### Dependency Relationships

```
Feature 8 (Embedder) is required by:
  - Feature 7 (CLI) for import/reembed commands
  - Feature 1 (Hybrid Retrieval) for vector search
  - Feature 6 (Auto-Capture) for embedding extracted memories

Feature 9 (Session Memory) is a consumer of:
  - Feature 8 (Embedder) for embedding session summaries
  - Feature 4 (Smart Extraction) for extracting summary content
  - Feature 1 (Store) for persistence

Feature 7 (CLI) provides utilities for:
  - Feature 4 upgrade path (migrate legacy -> smart format)
  - Cross-DB migration (reembed for A/B testing)
```
