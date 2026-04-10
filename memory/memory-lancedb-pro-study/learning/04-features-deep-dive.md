# Features Deep-Dive: memory-lancedb-pro

**Date:** 2026-03-27
**Repository:** `/Users/sheldon/Documents/claw/reference/memory-lancedb-pro`
**Synthesized from:** 4 feature batch research documents

---

## Executive Summary

memory-lancedb-pro is an OpenClaw plugin providing long-term memory capabilities for AI agents via LanceDB. The system combines vector similarity search with BM25 full-text retrieval, uses LLM-powered extraction for automatic memory formation, and implements Weibull decay for lifecycle management.

**12 features** organized in two tiers:
- **Core (6):** Hybrid retrieval, cross-encoder reranking, multi-scope isolation, smart extraction, lifecycle management, auto-capture/recall
- **Secondary (6):** Management CLI, multi-provider embedding, session memory, workspace boundaries, reflection system, long-context chunking

**Key architectural insight:** The system uses a "store first, think later" philosophy where memories are captured automatically, classified by LLM into semantic categories, and managed via decay algorithms rather than explicit user organization.

---

## Core Features (Tier 1)

### Feature 1: Hybrid Retrieval (Vector + BM25)

**What it does:** Combines semantic vector similarity search with BM25 full-text keyword matching to achieve both contextually aware and keyword-exact recall.

**How it works:**

The retrieval pipeline (`src/retriever.ts`) executes in 11 stages:

1. **Parallel Search** - Vector and BM25 searches run simultaneously via `Promise.all`
2. **RRF Fusion** - Results fused using weighted formula: `vectorScore * 0.7 + bm25Score * 0.3`
3. **Keyword Boost** - BM25 scores >= 0.75 receive a 0.92 boost to preserve exact keyword hits
4. **Ghost Entry Detection** - BM25-only results checked against vector store to skip deleted entries
5. **Score Filtering** - Minimum threshold (default 0.3) removes low-confidence results
6. **Reranking** - Optional cross-encoder refinement
7. **Recency Boost** - Exponential decay: `exp(-ageDays / halfLifeDays) * 0.1`
8. **Importance Weight** - Score *= `0.7 + 0.3 * importance`
9. **Length Normalization** - Penalizes long entries via logarithmic formula
10. **Time Decay** - Weibull-style decay with access reinforcement
11. **MMR Diversity** - Cosine similarity threshold (0.85) defers similar entries

**Key implementation details:**

- **Tag prefix detection:** Queries like `proj:AIF` bypass hybrid entirely, using BM25-only with mustContain filtering to prevent semantic false positives
- **Cross-process locking:** Uses `proper-lockfile` for atomic writes across multiple OpenClaw instances
- **Delete+Readd pattern:** LanceDB doesn't support in-place updates; the update method deletes then re-adds with rollback on failure
- **Lexical fallback:** If FTS index unavailable, falls back to substring matching across text, l0_abstract, l1_overview, l2_content with weighted scoring

**Notable patterns:**
```typescript
// Vector search over-fetch to account for filtering
const fetchMultiplier = filtering ? 20 : 10;
const maxFetch = Math.min(Math.round(topK * fetchMultiplier), 200);

// BM25 score normalization via sigmoid
const normalized = 1 / (1 + Math.exp(-rawScore / 5));
```

**Technical debt:**
- Hardcoded weights (0.7/0.3) not configurable at runtime
- Ghost entry detection adds `hasId()` call overhead for every BM25-only result
- Update race condition exists during delete-readd window

---

### Feature 2: Cross-Encoder Reranking

**What it does:** Post-retrieval reranking using cross-encoder models from multiple providers (Jina, SiliconFlow, Voyage AI, Pinecone, DashScope, TEI) with 60/40 blend of cross-encoder and original fused scores.

**How it works:**

1. Provider-specific request building adapts to each API's format
2. 5-second timeout via AbortController prevents stalling
3. Response parsing normalizes different score fields to unified format
4. Score blending: `blendedScore = item.score * 0.6 + original.score * 0.4`
5. Preservation floors prevent keyword hits from being demoted
6. Cosine similarity fallback if reranking fails

**Provider request shapes:**

| Provider | Request Format | Auth |
|----------|---------------|------|
| Jina/SiliconFlow | `{model, query, documents, top_n}` | Bearer |
| Voyage | `{model, query, documents, top_k}` | Bearer |
| Pinecone | `{model, query, documents: [{text}], top_n, rank_fields}` | Api-Key |
| DashScope | `{input: {query, documents}}` | Bearer |
| TEI | `{query, texts}` | Bearer |

**Preservation floors based on BM25:**
- >= 0.75: Exact keyword hits preserved (floor 1.0 or 0.95)
- >= 0.60: Moderate preservation (floor 0.95 or 0.9)
- < 0.60: More aggressive demotion (floor 0.8 or 0.5)

**Notable patterns:**
```typescript
// Abort controller timeout pattern
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 5000);
const response = await fetch(endpoint, { signal: controller.signal });
clearTimeout(timeout);

// Cosine similarity fallback
const combinedScore = result.score * 0.7 + cosineScore * 0.3;
```

**Technical debt:**
- Hardcoded blend weights not configurable
- No retry on transient failures (rate limits, 5xx)
- Provider-specific parsing is fragile with multiple fallback paths

---

### Feature 3: Multi-Scope Isolation

**What it does:** Memory isolation across `global`, `agent:<id>`, `project:<id>`, `user:<id>`, `custom:<name>`, and `reflection:agent:<id>` scopes with configurable ACL.

**How it works:**

**Scope patterns:**
```typescript
const SCOPE_PATTERNS = {
  GLOBAL: "global",
  AGENT: (agentId) => `agent:${agentId}`,
  CUSTOM: (name) => `custom:${name}`,
  REFLECTION: (agentId) => `reflection:agent:${agentId}`,
  PROJECT: (projectId) => `project:${projectId}`,
  USER: (userId) => `user:${userId}`,
};
```

**Access check flow:**
1. System bypass IDs (`"system"`, `"undefined"`) get full enumeration
2. Explicit ACL checked first from `config.agentAccess[agentId]`
3. Default returns `["global", "agent:{id}", "reflection:agent:{id}"]`
4. Reflection scope auto-granted even if not explicitly configured

**Store layer filter semantics:**
- `undefined`: No filtering (full bypass)
- `[]`: Explicit deny-all
- `["global", ...]`: Restrict to listed scopes

**ClawTeam extension:** Team-level scopes injected via environment variable `CLAWTEAM_MEMORY_SCOPE`, wrapping `getAccessibleScopes` to add extra scopes for all agents.

**Notable patterns:**
```typescript
// Session key parsing extracts agent ID
// "agent:main:discord:channel:123" -> "main"

// System bypass IDs
const SYSTEM_BYPASS_IDS = new Set(["system", "undefined"]);

// Always grant own reflection scope
function withOwnReflectionScope(scopes, agentId) {
  const reflectionScope = SCOPE_PATTERNS.REFLECTION(agentId);
  return scopes.includes(reflectionScope) ? [...scopes] : [...scopes, reflectionScope];
}
```

**Technical debt:**
- Monomorphic scope patterns - adding new types requires code changes
- No hierarchical scopes (no wildcards, no inheritance)
- Bypass ID `"undefined"` is a magic string that could cause issues
- No scope delegation (no transitive access)
- Global scope special-cased throughout

---

### Feature 4: Smart Memory Extraction

**What it does:** LLM-powered extraction into 6 semantic categories (profile, preferences, entities, events, cases, patterns) with L0/L1/L2 layered storage and two-stage deduplication.

**How it works:**

**Extraction pipeline:**
```
extractAndPersist(conversationText)
  -> extractCandidates()       [truncate, strip metadata, LLM call]
  -> batchDedup()              [cosine similarity pre-filter, O(n^2), n <= 5]
  -> processCandidate()       [per-survivor: vector search, admission, dedup]
```

**Dedup decisions:**
- `skip`: Duplicate, no action
- `create`: Store as new
- `merge`: LLM merge with existing (preferences/entities/patterns)
- `supersede`: Create new, invalidate old (temporal categories)
- `support`: Update context support stats
- `contextualize`: Create linked contextual variant
- `contradict`: Record contradiction evidence

**Three-level memory structure:**

| Level | Field | Purpose |
|-------|-------|---------|
| L0 | `l0_abstract` | One-line index for search/BM25 |
| L1 | `l1_overview` | Structured Markdown with category headings |
| L2 | `l2_content` | Full narrative with background |

**Six categories with dedup strategies:**

| Category | Strategy | Notes |
|----------|----------|-------|
| profile | ALWAYS_MERGE | User identity |
| preferences | MERGE/SUPERSEDE | Habits, likes/dislikes |
| entities | MERGE/SUPERSEDE | People, projects, orgs |
| events | APPEND_ONLY | Decisions, milestones |
| cases | APPEND_ONLY | Problem-solution pairs |
| patterns | MERGE | Reusable processes |

**Notable patterns:**
```typescript
// Two-stage dedup dramatically reduces LLM calls
// Stage 1: Fast cosine similarity pre-filter
if (similarity > 0.85) drop candidate;

// Stage 2: LLM decision only for survivors

// Envelope metadata stripping for weaker LLMs
stripEnvelopeMetadata()
// Removes: "System: [timestamp] Channel[account]..."
// Removes: "Conversation info (untrusted metadata):" + JSON
```

**Technical debt:**
- Profile category merges unconditionally without dedup - conflicting info merged without resolution
- No rollback on partial failures in batch processing
- LLM merge output not validated against inputs
- Hardcoded thresholds (0.7 for vector search, 0.85 for batch dedup)
- No extraction timeout

---

### Feature 5: Memory Lifecycle Management

**What it does:** Weibull decay engine with composite scoring (recency + frequency + importance) and three-tier promotion system (Peripheral, Working, Core).

**How it works:**

**Composite score formula:**
```
Composite = 0.4 * recency + 0.3 * frequency + 0.3 * intrinsic
```

**Recency (Weibull stretched-exponential):**
```typescript
effectiveHL = halfLife * exp(1.5 * importance)
lambda = ln(2) / effectiveHL
recency = exp(-lambda * daysSince^beta)

// Beta varies by tier:
// Core: beta = 0.8 (slower decay)
// Working: beta = 1.0 (standard)
// Peripheral: beta = 1.3 (faster decay)
```

**Frequency (logarithmic saturation):**
```typescript
base = 1 - exp(-accessCount / 5)
recentBonus = exp(-avgGapDays / 30)  // only if accessCount > 1
frequency = base * (0.5 + 0.5 * recentBonus)
```

**Tier promotion rules:**

| Transition | Condition |
|------------|-----------|
| Peripheral -> Working | accessCount >= 3 AND composite >= 0.4 |
| Working -> Core | accessCount >= 10 AND composite >= 0.7 AND importance >= 0.8 |
| Working -> Peripheral | composite < 0.15 OR (age > 60 days AND accessCount < 3) |
| Core -> Working | composite < 0.15 AND accessCount < 3 |

**Access tracker design:**
- Debounced batch write-back (5s default)
- Access count decays with 30-day half-life
- Effective half-life = `baseHL + baseHL * reinforcementFactor * log1p(effectiveAccessCount)`
- Hard cap on multiplier (5x default)

**Notable patterns:**
```typescript
// Importance-modulated half-life
effectiveHL = halfLife * exp(1.5 * importance);
// importance 0.7 -> effectiveHL ~= 86 days
// importance 0.9 -> effectiveHL ~= 110 days

// Tier-specific decay curves
const TIER_BETAS = { core: 0.8, working: 1.0, peripheral: 1.3 };
```

**Technical debt:**
- Age-based demotion uses calendar days - memories with identical access patterns but different creation times decay differently
- Access count is a proxy metric - quality of access not measured
- No distributed coordination - multiple instances could lose increments on flush conflicts
- Core -> Working demotion threshold is very aggressive (composite < 0.15 AND accessCount < 3)
- No cap on Core/Working memory count

---

### Feature 6: Auto-Capture & Auto-Recall

**What it does:** Automatic memory extraction on `agent_end` hook and context injection via `before_prompt_build` hook. Filters noise and adaptively skips retrieval for non-memory queries.

**How it works:**

**Noise filter (`noise-filter.ts`):**

| Category | Patterns |
|----------|----------|
| Denials | "I don't have any information", "I don't recall" |
| Meta-questions | "Do you remember X?", "Did I tell you about Y?" |
| Boilerplate | "hi", "hello", "fresh session", "HEARTBEAT" |
| Diagnostic artifacts | "query -> none", "no explicit solution" |

**Adaptive retrieval decision tree (`adaptive-retrieval.ts`):**
```
shouldSkipRetrieval(query)
  -> FORCE_RETRIEVE_PATTERNS matched? -> ALWAYS retrieve
  -> Length < 5 chars? -> skip
  -> SKIP_PATTERNS matched (greetings, slash commands, affirmations)? -> skip
  -> CJK text < 6 chars (without ?)? -> skip
  -> Non-CJK text < 15 chars (without ?)? -> skip
  -> Default: retrieve
```

**OpenClaw metadata stripping:**
- Strips `[cron:...]` wrappers
- Strips `Conversation info (untrusted metadata):` JSON blocks
- Strips `Sender (untrusted metadata):` JSON blocks
- Strips timestamp prefixes `[Mon 2026-03-02 04:21 GMT+8]`
- Strips `@mention` addressing prefixes

**Auto-capture cleanup (`auto-capture-cleanup.ts`):**
- Multi-pass metadata stripping (up to 6 iterations until stable)
- Collapses excessive newlines (3+ -> 2)

**Notable patterns:**
```typescript
// CJK-aware length threshold
const hasCJK = /[\u4e00-\u9fff\u3040-\u309f\u30a0-\u30ff\uac00-\ud7af]/.test(trimmed);
const defaultMinLength = hasCJK ? 6 : 15;

// Force-retrieve override before length check
// "你记得吗" (do you remember) ALWAYS retrieves
```

**Technical debt:**
- Pattern-based detection is brittle - new phrasing not caught
- No cross-session deduplication - same memory captured multiple times
- Metadata stripping relies on string matching - format changes break stripping
- No retrieval result caching
- Force-retrieve patterns are English-centric

---

## Secondary Features (Tier 2)

### Feature 7: Management CLI

**What it does:** Full CLI toolkit via `openclaw memory-pro` providing list, search, stats, delete, delete-bulk, export, import, reembed, upgrade, migrate, auth, and reindex-fts commands.

**Key implementation (`cli.ts`, 1,350 lines):**

- Commander-based command registration
- Context-driven architecture: `CLIContext` injected with store, retriever, scopeManager, migrator, embedder, llmClient
- Factory function: `createMemoryCLI(context)` returns registration function

**Command inventory:**

| Command | Lines | Purpose |
|---------|-------|---------|
| list | 649-691 | List with scope/category filtering, pagination |
| search | 693-736 | Hybrid retrieval with scoring |
| stats | 738-789 | Aggregate statistics |
| delete | 791-815 | Single deletion with scope access control |
| delete-bulk | 817-856 | Bulk delete with --dry-run, date filtering |
| export | 858-905 | JSON export excluding vectors |
| import | 907-1037 | Idempotent import (id-based + 0.95 similarity dedupe) |
| reembed | 1039-1162 | Re-embed from source to target DB |
| upgrade | 1164-1229 | Legacy to 6-category smart format |
| migrate | 1231-1319 | Migration workflow |
| auth | 468-647 | OAuth login/status/logout |
| reindex-fts | 1321-1340 | Rebuild BM25 index |
| version | 460-466 | Print version |

**Notable patterns:**
```typescript
// OAuth backup/restore pattern
// On login: backup existing LLM config
// On logout: restore from backup or build fallback

// Import idempotency
// 1. If import has id and target DB has it -> skip
// 2. If no id, similarity check (score > 0.95) -> skip

// Re-embed safety
// Prevents in-place re-embedding
// Uses fs.realpath to resolve symlinks before comparison
// Requires --force flag to override
```

**Technical debt:**
- OAuth logout fallback may not represent original API-key endpoint
- Import similarity threshold (0.95) may miss semantically similar memories
- Re-embed loads entire source DB into memory (no streaming)
- No progress indicator for long imports

---

### Feature 8: Multi-Provider Embedding

**What it does:** Compatible with any OpenAI-compatible API provider (Jina, OpenAI, Voyage, Google Gemini, Ollama) with multi-key rotation, retry logic, and auto-chunking.

**Key implementation (`src/embedder.ts`, 942 lines):**

**Provider detection via regex:**
```typescript
/ api\.openai\.com/ -> "openai"
/ api\.jina\.ai/ or /^jina-/ -> "jina"
/ api\.voyageai\.com/ or /^voyage/ -> "voyage-compatible"
/ api\.openai\.azure\.com/ -> "azure-openai"
otherwise -> "generic-openai-compatible"
```

**Capability mapping:**

| Provider | encoding_format | normalized | taskField | dimensionsField |
|----------|----------------|------------|----------|----------------|
| openai | float | false | null | "dimensions" |
| jina | true | true | "task" | "dimensions" |
| voyage-compatible | false | false | "input_type" | "output_dimension" |

**Multi-key round-robin:**
```typescript
private nextClient(): OpenAI {
  const client = this.clients[this._clientIndex % this.clients.length];
  this._clientIndex = (this._clientIndex + 1) % this.clients.length;
  return client;
}
```

**Rate limit handling with key rotation:**
- Tries each key at most once before giving up
- On 429/503 or `rate_limit_exceeded`/`insufficient_quota` codes -> rotate
- Other errors -> propagate immediately

**Auto-chunking for long documents:**
- Triggered on context length errors
- Recursion depth limit: MAX_EMBED_DEPTH = 3
- Single chunk output detection: if chunk ~90% of original, force-truncate to 50%
- Averages embeddings across chunks

**Embedding cache (LRU + TTL):**
- SHA256 hash of `${task}:${text}`, truncated to 24 chars
- 256 entries, 30 min TTL
- LRU eviction

**Task-aware API:**
```typescript
embedQuery(text)       // Uses "retrieval.query" task hint
embedPassage(text)     // Uses "retrieval.passage" task hint
embedBatchQuery()      // Parallel batch with query task
embedBatchPassage()    // Parallel batch with passage task
```

**Technical debt:**
- EMBEDDING_DIMENSIONS table manually maintained - no dynamic discovery
- Cache key ignores API key - same text+task on different keys produces same hit
- No circuit breaker for failing providers

---

### Feature 9: Session Memory

**What it does:** Session summarization on `/new` command with configurable message count. Stores session summaries to LanceDB for cross-session continuity.

**Key implementation (`src/session-compressor.ts`, 332 lines):**

**Scoring algorithm for text segments:**

| Score | Reason | Pattern |
|-------|--------|---------|
| 1.0 | tool_call | tool_use, tool_result, function_call, memory operations |
| 0.95 | correction | actually, instead, wrong |
| 0.85 | decision | let's go with, confirmed, approved |
| 0.3 | system_xml | XML tags with content |
| 0.7 | substantive | >80 chars (>30 for CJK), not system XML |
| 0.5 | short_question | Contains ? |
| 0.4 | short_statement | Short non-question text |
| 0.1 | acknowledgment | ok, okay, thanks |
| 0.0 | empty | Whitespace only |

**Compression algorithm:**
```
1. If all texts fit in budget -> return all
2. Always keep first and last text (session boundaries)
3. Score remaining texts, sort by score desc, then index asc
4. Identify paired tool_call + tool_result (adjacent indices)
5. Greedily select highest-scoring until budget exhausted
6. Re-sort selected by original index (chronological order)
```

**Conversation value estimation:**
```
Signal = memory_intent(0.5) + tool_calls(0.4) + corrections(0.3) + substantive(0.2) + multi_turn(0.1)
Cap: 1.0
```

**Session recovery:**
- Discovers session directories from: explicit paths, environment/config, agent ID discovery
- Cross-product: for each `(home, agentId)` pair, adds `home/agents/<agentId>/sessions`

**Technical debt:**
- CJK character threshold (30 chars) is approximate, not validated
- Paired tool_call detection only looks at adjacent indices - fragile
- Session recovery may find stale directories
- Magic numbers in value estimation not configurable
- Session summaries lose temporal ordering

---

### Feature 10: Workspace Boundary & Privacy

**What it does:** Privacy controls preventing cross-workspace memory contamination with configurable boundary rules for user profile/name/addressing data.

**Key implementation:**

**Workspace boundary (`workspace-boundary.ts`):**
- Memory classification via `isUserMdExclusiveMemory()`
- Uses `classifyIdentityAndAddressingMemory()` for name/addressing slots
- Filters recall results when enabled

**Admission control (`admission-control.ts`):**

**Scoring components (weighted sum):**

| Component | Weight | Description |
|-----------|--------|-------------|
| typePrior | 0.6 | Category-specific prior |
| utility | 0.1 | LLM-evaluated future usefulness |
| confidence | 0.1 | Support from conversation context |
| novelty | 0.1 | Dissimilarity to existing memories |
| recency | 0.1 | Gap since last similar memory |

**Type priors:**
```
profile: 0.95, preferences: 0.9, entities: 0.75, cases: 0.8, patterns: 0.85, events: 0.45
```

**Decision logic:**
```
score = weightedSum(featureScores)
decision = score < rejectThreshold ? "reject" : "pass_to_dedup"
hint = score >= admitThreshold && novelty < 0.55 ? "add" : "update_or_merge"
```

**Notable patterns:**
```typescript
// CJK tokenization preserves semantics
const isHanChar = /\p{Script=Han}/u;  // May not work in older Node

// LCS-based similarity for confidence
rougeLikeF1()  // Longest Common Subsequence
```

**Technical debt:**
- Admission controller is sync but utility scoring is async - no parallel execution
- Hardcoded 0.55 novelty threshold magic number
- CJK handling uses `/p{Script=Han}/u` which may not work in older Node versions
- No dedup step in admission (pass_to_dedup comment but actual dedup post-admission)

---

### Feature 11: Reflection System

**What it does:** Learning from agent behavior patterns with reflection slices, event stores, and governance metadata for self-improvement.

**Key implementation:**

**Reflection storage pipeline:**
```
reflectionText
  -> extractInjectableReflectionSlices()
  -> ReflectionSlices { invariants: string[], derived: string[] }
  -> buildReflectionEvent/Item/Legacy payloads
  -> vector search dedup (similarity > 0.97 skip)
  -> embed & store
```

**Slice sanitization:**
```typescript
// Injection prevention patterns
/^\s*(?:ignore|disregard|forget|override|bypass)\b.../i  // Override attempts
/\b(?:reveal|print|dump|show)\b...\b(system prompt|secrets?)\b/i  // Extraction
/<[\s]*\/?\s*(?:system|assistant|user|tool).../i  // XML tag injection
/^(?:system|assistant|user|developer|tool)\s*:/i  // Role play prefixes
```

**Reflection ranking (logistic decay):**

| Type | Midpoint (days) | k | Base Weight |
|------|----------------|---|-------------|
| Invariant | 45 | 0.22 | 1.1 |
| Derived | 7 | 0.65 | 1.0 |
| Mapped: decision | 45 | 0.25 | 1.1 |
| Mapped: user-model | 21 | 0.3 | 1.0 |
| Mapped: agent-model | 10 | 0.35 | 0.95 |
| Mapped: lesson | 7 | 0.45 | 0.9 |

**Intent analyzer for adaptive recall:**
```typescript
// Pure rule-based, no LLM
const INTENT_RULES = {
  preference: { patterns: [prefer, like, dislike], categories: [preference, decision] },
  decision: { patterns: [why did, decision, chose], categories: [decision, fact] },
  entity: { patterns: [who is, tell me about], categories: [entity, fact] },
  event: { patterns: [when did, what happened], categories: [entity, decision] },
  fact: { patterns: [how does, what is, explain], categories: [fact, entity] },
};
```

**Technical debt:**
- Reflection schema version 4 - migration path for older entries unclear
- Max age per type not enforced during ranking (only during loading)
- Line 273-274 bug: fallback logic for derived may not match legacy format
- No source validation for reflection path
- Intent analyzer bilingual approach may become unwieldy

---

### Feature 12: Long-Context Chunking

**What it does:** Intelligent chunking for processing long documents triggered when embedding providers throw context-length errors.

**Key implementation (`chunker.ts`):**

**Public API (`chunkDocument`):**
```
while pos < text.length:
  1. If remaining <= maxChunkSize: Take remainder
  2. Otherwise:
     a. findSplitEnd() at semantic boundary
     b. sliceTrimWithIndices() to extract and trim
     c. If trimmed < minChunkSize: Hard split fallback
     d. Push chunk
     e. pos = end - overlapSize (move forward with overlap)
```

**Smart chunking (`smartChunk`):**
```typescript
limit = EMBEDDING_CONTEXT_LIMITS[embedderModel] ?? 8192

// CJK adaptation
if (cjkRatio > 0.3):
  divisor = 2.5  // CJK chars ~2-3 tokens each
else:
  divisor = 1

// Conservative sizing
maxChunkSize = floor(base * 0.7 / divisor)   // 70% of limit
overlapSize = floor(base * 0.05 / divisor)     // 5% of limit
minChunkSize = floor(base * 0.1 / divisor)     // 10% of limit
```

**Split point finding priority:**
1. Line limit enforcement
2. Semantic split (sentence-ending punctuation)
3. Newline boundary
4. Whitespace boundary

**CJK detection:**
```typescript
const CJK_RE = /[\u3040-\u309F\u30A0-\u30FF\u3400-\u4DBF\u4E00-\u9FFF\uAC00-\uD7AF\uF900-\uFAFF]/
cjkRatio = cjkChars / nonWhitespaceChars
```

**Supported models with limits:**

| Model | Context | Chunk (70%) | Overlap (5%) |
|-------|---------|------------|--------------|
| jina-embeddings-v5-text-small | 8192 | 5734 | 409 |
| text-embedding-3-small | 8192 | 5734 | 409 |
| gemini-embedding-001 | 2048 | 1433 | 102 |
| all-MiniLM-L6-v2 | 512 | 358 | 40 |

**Technical debt:**
- Character count as token proxy - conservative but imprecise
- Hardcoded magic numbers (0.7, 0.05, 0.1, 0.3, 2.5) without named constants
- No hierarchical chunking planned for future
- Guard loop could get stuck with very small advances
- Single document hash for caching - minor edits trigger full recompute

---

## Feature Interactions and Dependencies

### Dependency Graph

```
Feature 8 (Embedder)
  └── Required by: 1, 4, 6, 7, 9

Feature 1 (Hybrid Retrieval)
  └── Uses: 8 (embedder), store (persistence)
  └── Required by: 2 (reranking)

Feature 4 (Smart Extraction)
  └── Uses: 8 (embedder), store (persistence)
  └── Produces: categorized memories
  └── Required by: 9 (session memory)

Feature 5 (Lifecycle)
  └── Uses: access tracking, decay engine
  └── Affects: retrieval scoring (applyTimeDecay)

Feature 6 (Auto-Capture/Recall)
  └── Uses: 4 (extraction), 1 (retrieval)
  └── Hooks: agent_end, before_prompt_build

Feature 7 (CLI)
  └── Uses: 8, store, scopes
  └── Commands: import, reembed, upgrade, migrate

Feature 9 (Session Memory)
  └── Uses: 8 (embedder), 4 (extraction), store
  └── Integration: command:new, command:reset hooks

Feature 10 (Workspace Boundaries)
  └── Affects: retrieval filtering, admission control
  └── Independent of other features

Feature 11 (Reflection)
  └── Uses: 8 (embedder), store
  └── Integration: session boundaries, agent reflection

Feature 12 (Long-Context Chunking)
  └── Triggered by: embedder context-length errors
  └── Independent design
```

### Cross-Feature Patterns

**1. Configuration-Driven Design**
All features rely on configuration objects merged with defaults. This allows runtime customization but creates complex validation logic (seen in scopes, embedder, admission control).

**2. Error Handling Philosophy**
- **Fail-open for retrieval:** Lexical fallback, cosine fallback for reranking
- **Fail-closed for isolation:** Invalid scopes denied, system bypass only for internal tasks
- **Fail-open for extraction:** No extraction timeout, weak LLMs store metadata verbatim

**3. CJK-Aware Processing**
Multiple features (session compression, chunking, adaptive retrieval, admission control) implement CJK character detection and adjusted thresholds because CJK characters carry more semantic density.

**4. Two-Stage Processing**
- Smart extraction: Cosine pre-filter then LLM decision
- Admission control: Fast scoring then async utility scoring
- Reranking: Primary provider then cosine fallback

---

## Key Architectural Decisions

### 1. LanceDB as Primary Store
Using LanceDB provides both vector search (ANN) and FTS indexing in a single database, simplifying the stack. The delete-readd pattern for updates acknowledges LanceDB's immutability model.

### 2. L0/L1/L2 Layered Metadata
Storing abstract/overview/content as separate metadata fields rather than separate records enables flexible retrieval without duplication. L0 serves as the BM25 text field.

### 3. Category-Specific Dedup Strategies
Rather than a universal dedup algorithm, each category has its own strategy (ALWAYS_MERGE, APPEND_ONLY, MERGE/SUPERSEDE) reflecting different memory semantics.

### 4. Hook-Based Integration
Auto-capture and auto-recall integrate via OpenClaw hooks (`agent_end`, `before_prompt_build`) rather than explicit API calls, making memory transparent to users.

### 5. Decay as Primary Lifecycle
Rather than explicit deletion, the system uses Weibull decay with composite scoring. Memories fade naturally but can be reinforced through access.

### 6. Admission Control Before Dedup
The gating decision happens before expensive dedup, preventing low-quality candidates from consuming LLM resources.

### 7. Reflection as Separate Scope
Reflection memories stored in `reflection:agent:{id}` scope, isolated from regular memories but accessible to the owning agent.

---

## Most Valuable Patterns to Steal

### 1. Two-Stage Dedup (Feature 4)
**Pattern:** Fast pre-filter (cosine similarity) to reduce expensive operations (LLM calls)
**Applicability:** Any system where expensive dedup can be avoided for obvious duplicates
**Code location:** `src/smart-extractor.ts:803-830`, `src/batch-dedup.ts`

### 2. Multi-Key Round-Robin with Rate Limit Handling (Feature 8)
**Pattern:** Maintain multiple API keys, rotate on 429/rate_limit_exceeded, fail-fast on other errors
**Applicability:** Any multi-key API integration with rate limits
**Code location:** `src/embedder.ts:483-555`

### 3. Debounced Batch Write-Back (Feature 5)
**Pattern:** Synchronous in-memory updates with deferred I/O, batched writes
**Applicability:** High-frequency read, low-frequency write scenarios
**Code location:** `src/access-tracker.ts:142-180`

### 4. Weibull Decay with Tier-Specific Beta (Feature 5)
**Pattern:** Different decay curves per importance tier
**Applicability:** Memory systems where some facts should persist longer
**Code location:** `src/decay-engine.ts:90-130`

### 5. CJK-Aware Length Thresholds (Features 6, 9, 12)
**Pattern:** Detect CJK script, apply character-based thresholds adjusted for semantic density
**Applicability:** Any multilingual text processing
**Code location:** `src/noise-filter.ts`, `src/chunker.ts:171-184`

### 6. Preservation Floors in Score Blending (Feature 2)
**Pattern:** When blending scores, preserve floor based on BM25 or other signal
**Applicability:** Reranking systems where keyword hits should not be demoted
**Code location:** `src/retriever.ts:961-973`

### 7. Cross-Process Locking (Feature 1)
**Pattern:** `proper-lockfile` for atomic operations across processes
**Applicability:** Any multi-process data store
**Code location:** `src/store.ts:205-217`

### 8. Injection Prevention via Pattern Matching (Feature 11)
**Pattern:** Extensive regex patterns to filter unsafe content before injection
**Applicability:** Any system that echoes back user/agent content as prompts
**Code location:** `src/reflection-slices.ts:210-218`

### 9. Session Boundary Preservation (Feature 9)
**Pattern:** Always keep first and last message, greedily select highest-scoring intermediate
**Applicability:** Summarization with limited budget
**Code location:** `src/session-compressor.ts:171-277`

### 10. Adaptive Intent Routing (Feature 11)
**Pattern:** Pure rule-based intent detection to minimize latency, category boosting for retrieval
**Applicability:** Auto-recall systems where LLM calls should be avoided
**Code location:** `src/intent-analyzer.ts:58-196`

---

## Summary Table

| Feature | Primary File | Complexity | External I/O | Configurability |
|---------|-------------|------------|--------------|-----------------|
| Hybrid Retrieval | retriever.ts | High | None | Medium |
| Cross-Encoder | retriever.ts | Medium | Rerank API | Low |
| Multi-Scope | scopes.ts | Medium | None | High |
| Smart Extraction | smart-extractor.ts | Very High | LLM API | Low |
| Lifecycle | decay-engine.ts | High | None | Medium |
| Auto-Capture | noise-filter.ts | Low | None | Medium |
| Management CLI | cli.ts | High | None | High |
| Embedder | embedder.ts | High | Embedding API | Medium |
| Session Memory | session-compressor.ts | Medium | None | Medium |
| Workspace | workspace-boundary.ts | Medium | None | High |
| Reflection | reflection-store.ts | High | LLM API | Medium |
| Chunking | chunker.ts | Medium | None | Low |

---

*Generated by synthesizing 4 feature batch deep-dive documents*
