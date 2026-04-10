# OpenViking Feature Research: Batch 1 (Features 1-3)

**Project:** OpenViking - Context Database for AI Agents
**Repository:** `/Users/sheldon/Documents/claw/reference/OpenViking`
**Research Date:** 2026-03-27
**Features Covered:** Virtual Filesystem (viking:// Protocol), Tiered Context Loading (L0/L1/L2), Directory Recursive Retrieval

---

## Feature 1: Virtual Filesystem (viking:// Protocol)

### Overview

The Virtual Filesystem (VikingFS) is the core abstraction layer that unifies all context management under a filesystem paradigm. Every piece of information in OpenViking is identified by a `viking://` URI and can be accessed through standard filesystem-like operations.

### Core Implementation

**Primary File:** `openviking/storage/viking_fs.py` (1910 lines)

The `VikingFS` class wraps an AGFS (Abstract Google File System) client and provides:
- URI-to-path mapping with account isolation
- L0/L1 reading (abstract, overview)
- Relation management
- Semantic search integration
- Vector store synchronization

### URI System

**URI Format:** `viking://{scope}/{path}`

**Valid Scopes (from `openviking_cli/utils/uri.py`):**
```python
VALID_SCOPES = {"resources", "user", "agent", "session", "queue", "temp"}
```

**Key URI Operations:**

```python
# Path to URI conversion (lines 1198-1220)
def _path_to_uri(self, path: str, ctx: Optional[RequestContext] = None) -> str:
    # /local/{account}/... -> viking://...
    if path.startswith("/local/"):
        inner = path[7:].strip("/")
        parts = [p for p in inner.split("/") if p]
        if parts and parts[0] == real_ctx.account_id:
            parts = parts[1:]
        return f"viking://{'/'.join(parts)}"

# URI to path conversion (lines 1168-1181)
def _uri_to_path(self, uri: str, ctx: Optional[RequestContext] = None) -> str:
    # viking://{remainder} -> /local/{account_id}/{remainder}
    safe_parts = [self._shorten_component(p, self._MAX_FILENAME_BYTES) for p in parts]
    return f"/local/{account_id}/{'/'.join(safe_parts)}"
```

### Security Features

**URI Traversal Protection (lines 253-271):**
```python
@classmethod
def _normalized_uri_parts(cls, uri: str) -> tuple[str, List[str]]:
    normalized = cls._normalize_uri(uri)
    parts = [p for p in normalized[len("viking://") :].strip("/").split("/") if p]

    for part in parts:
        if part in {".", ".."}:
            raise PermissionError(f"Unsafe URI traversal segment '{part}' in {normalized}")
        if "\\" in part:
            raise PermissionError(f"Unsafe URI path separator '\\\\' in component...")
        if len(part) >= 2 and part[1] == ":" and part[0].isalpha():
            raise PermissionError(f"Unsafe URI drive-prefixed component...")
```

**Access Control (lines 1245-1267):**
```python
def _is_accessible(self, uri: str, ctx: RequestContext) -> bool:
    if ctx.role == Role.ROOT:
        return True
    scope = parts[0]
    if scope in {"resources", "temp"}:
        return True
    if scope == "_system":
        return False
    # Space-based isolation for user/agent/session scopes
```

### Account Isolation

The filesystem uses account-isolated paths:
- Path format: `/local/{account_id}/{rest}`
- Each account's data is isolated under its own directory
- The `_ls_entries` method uses whitelisting at account root level

```python
def _ls_entries(self, path: str) -> List[Dict[str, Any]]:
    entries = self.agfs.ls(path)
    parts = [p for p in path.strip("/").split("/") if p]
    # At account root, use VALID_SCOPES whitelist
    if len(parts) == 2 and parts[0] == "local":
        return [e for e in entries if e.get("name") in VikingURI.VALID_SCOPES]
    # Elsewhere, filter internal names
    return [e for e in entries if e.get("name") not in self._INTERNAL_NAMES]
```

### File Operations

**Read with Encryption Support (lines 281-320):**
```python
async def read(self, uri: str, offset: int = 0, size: int = -1, ctx: Optional[RequestContext] = None) -> bytes:
    self._ensure_access(uri, ctx)
    path = self._uri_to_path(uri, ctx=ctx)

    if self._encryptor:
        # Must read entire file for decryption
        result = self.agfs.read(path, 0, -1)
        raw = await self._decrypt_content(raw, ctx=ctx)
        # Apply slicing on decrypted plaintext
        if offset > 0 or size != -1:
            raw = raw[offset:offset+size] if size != -1 else raw[offset:]
    else:
        result = self.agfs.read(path, offset, size)
```

**Write with Parent Directory Creation (lines 322-335):**
```python
async def write(self, uri: str, data: Union[bytes, str], ctx: Optional[RequestContext] = None) -> str:
    self._ensure_access(uri, ctx)
    path = self._uri_to_path(uri, ctx=ctx)
    if isinstance(data, str):
        data = data.encode("utf-8")
    data = await self._encrypt_content(data, ctx=ctx)
    return self.agfs.write(path, data)
```

### Relation Management

Relations are stored in `.relations.json` files:

```python
@dataclass
class RelationEntry:
    id: str
    uris: List[str]
    reason: str = ""
    created_at: str = field(default_factory=get_current_timestamp)
```

**Link operation (lines 1076-1100):**
```python
async def link(self, from_uri: str, uris: Union[str, List[str]], reason: str = "", ctx: Optional[RequestContext] = None):
    entries = await self._read_relation_table(from_path, ctx=ctx)
    link_id = next(f"link_{i}" for i in range(1, 10000) if f"link_{i}" not in existing_ids)
    entries.append(RelationEntry(id=link_id, uris=uris, reason=reason))
    await self._write_relation_table(from_path, entries, ctx=ctx)
```

### Search Integration

VikingFS integrates with `HierarchicalRetriever` for semantic search:

```python
async def find(self, query: str, target_uri: str = "", limit: int = 10, ...):
    retriever = HierarchicalRetriever(storage=storage, embedder=embedder, rerank_config=self.rerank_config)
    typed_query = TypedQuery(query=query, context_type=context_type, intent="", target_directories=[target_uri] if target_uri else None)
    result = await retriever.retrieve(typed_query, ctx=real_ctx, limit=limit, score_threshold=score_threshold, scope_dsl=filter)
```

### Clever Solutions

1. **Filename byte limit handling (lines 1149-1163):** Components are shortened if UTF-8 encoding exceeds 255 bytes using SHA256 hash suffix
2. **Context variable binding (lines 179-181, 237-244):** Uses Python's `contextvars` for thread-safe request context propagation
3. **Batch abstract fetching (lines 629-658):** Parallel fetch with semaphore limit of 6 concurrent operations
4. **Timestamp microsecond truncation (lines 1737-1744):** Handles 7+ digit microseconds in AGFS responses

### Technical Debt / Concerns

1. **Synchronous AGFS calls in async methods:** `agfs.mkdir()`, `agfs.read()`, `agfs.write()` are synchronous but called from async methods - no async variants
2. **Idempotent rm (lines 359-406):** Non-existent paths succeed after cleaning orphan index records - unusual but intentional
3. **mv implemented as cp+rm (lines 408-488):** Avoids lock file carryover but doubles I/O for large files
4. **Graceful fallback for AGFS read results:** Multiple checks for bytes vs objects with `.content` attribute (lines 1269-1282)

---

## Feature 2: Tiered Context Loading (L0/L1/L2)

### Overview

OpenViking uses a three-tier information model to balance retrieval efficiency and content completeness. The tiers are automatically generated for directories and represent different levels of summarization.

### Layer Definitions

| Layer | Name | File | Token Limit | Purpose |
|-------|------|------|-------------|---------|
| **L0** | Abstract | `.abstract.md` | ~100 tokens | Vector search, quick filtering |
| **L1** | Overview | `.overview.md` | ~2KB | Rerank, content navigation |
| **L2** | Detail | Original files | Unlimited | Full content, on-demand |

### Implementation in VikingFS

**L0 Abstract Reading (lines 786-804):**
```python
async def abstract(self, uri: str, ctx: Optional[RequestContext] = None) -> str:
    """Read directory's L0 summary (.abstract.md)."""
    self._ensure_access(uri, ctx)
    path = self._uri_to_path(uri, ctx=ctx)
    info = self.agfs.stat(path)
    if not info.get("isDir"):
        raise ValueError(f"{uri} is not a directory")
    file_path = f"{path}/.abstract.md"
    content_bytes = self._handle_agfs_read(self.agfs.read(file_path))
    if self._encryptor:
        content_bytes = await self._encryptor.decrypt(real_ctx.account_id, content_bytes)
    return self._decode_bytes(content_bytes)
```

**L1 Overview Reading (lines 806-824):** Identical pattern to abstract but reads `.overview.md`

**Batch Reading (lines 1541-1556):**
```python
async def read_batch(self, uris: List[str], level: str = "l0", ctx: Optional[RequestContext] = None) -> Dict[str, str]:
    results = {}
    for uri in uris:
        try:
            content = ""
            if level == "l0":
                content = await self.abstract(uri, ctx=ctx)
            elif level == "l1":
                content = await self.overview(uri, ctx=ctx)
            results[uri] = content
        except Exception:
            pass
    return results
```

### Semantic Processing Pipeline

When content is added, the SemanticProcessor generates L0/L1 bottom-up:

```
Leaf nodes → Parent directories → Root
```

Child directory L0s are aggregated into parent L1 for hierarchical navigation.

### Session Compression Integration

The `SessionCompressor` (`openviking/session/compressor.py`) handles memory extraction:

```python
class SessionCompressor:
    """Session memory extractor with 6-category memory extraction."""

    async def _index_memory(self, memory: Context, ctx: RequestContext, change_type: str = "added") -> bool:
        # For long memories, split into chunks for better retrieval precision
        if vectorize_text and len(vectorize_text) > semantic.memory_chunk_chars:
            chunks = self._chunk_text(vectorize_text, semantic.memory_chunk_chars, semantic.memory_chunk_overlap)
            for i, chunk in enumerate(chunks):
                chunk_memory.uri = f"{memory.uri}#chunk_{i:04d}"
                chunk_memory.parent_uri = memory.uri
                chunk_memory.set_vectorize(Vectorize(text=chunk))
                chunk_msg = EmbeddingMsgConverter.from_context(chunk_memory)
                await self.vikingdb.enqueue_embedding_msg(chunk_msg)
```

### Chunking Strategy (lines 196-218+)

Long memories are split into overlapping chunks:
- Each chunk gets `#chunk_NNNN` suffix in URI
- Parent pointer set to original memory URI
- Overlap ensures context continuity

### L0/L1 File Naming Convention

Special hidden files in each directory:
- `.abstract.md` - L0 (~100 tokens summary)
- `.overview.md` - L1 (~2KB overview)
- `.relations.json` - Related resources
- `.meta.json` - Metadata

### Documentation

From `docs/en/concepts/03-context-layers.md`:
- L0: Ultra-short, max ~100 tokens, quick perception
- L1: Moderate length ~1k tokens, navigation guide with section pointers
- L2: Full content, no token limit, on-demand loading

### Notable Implementation Details

1. **LLM-based deduplication** (`openviking/session/memory_deduplicator.py`): Before storing new memories, checks for duplicates using LLM
2. **6-category extraction**: PROFILE, PREFERENCES, ENTITIES, PATTERNS, TOOLS, SKILLS
3. **Memory chunking** with overlap for long content
4. **Pending semantic changes batched** and flushed after memory operations complete

---

## Feature 3: Directory Recursive Retrieval

### Overview

The hierarchical retriever combines vector search with directory traversal. It uses a priority queue to explore directories, propagating scores upward, with optional reranking for refined results.

### Core Implementation

**Primary File:** `openviking/retrieve/hierarchical_retriever.py` (619 lines)

### Retrieval Flow

```
Step 1: Determine starting directories by context_type
Step 2: Global vector search to locate initial directories
Step 3: Merge starting points + Rerank scoring
Step 4: Recursive search (priority queue)
Step 5: Convert to MatchedContext
```

### Key Classes and Constants

```python
class HierarchicalRetriever:
    MAX_CONVERGENCE_ROUNDS = 3  # Stop after multiple rounds with unchanged topk
    MAX_RELATIONS = 5  # Maximum relations per resource
    SCORE_PROPAGATION_ALPHA = 0.5  # Score propagation coefficient
    DIRECTORY_DOMINANCE_RATIO = 1.2  # Directory score must exceed max child score
    GLOBAL_SEARCH_TOPK = 5  # Global retrieval count
    HOTNESS_ALPHA = 0.2  # Weight for hotness score in final ranking
    LEVEL_URI_SUFFIX = {0: ".abstract.md", 1: ".overview.md"}
```

### Main Retrieval Method (lines 87-222)

```python
async def retrieve(self, query: TypedQuery, ctx: RequestContext, limit: int = 5,
                   mode: str = RetrieverMode.THINKING, score_threshold: Optional[float] = None,
                   score_gte: bool = False, scope_dsl: Optional[Dict[str, Any]] = None) -> QueryResult:
    t0 = time.monotonic()
    effective_threshold = score_threshold if score_threshold is not None else self.threshold

    # Create proxy wrapper with current ctx
    vector_proxy = VikingDBManagerProxy(self.vector_store, ctx)

    # Generate query vectors once
    if self.embedder:
        result: EmbedResult = self.embedder.embed(query.query, is_query=True)
        query_vector = result.dense_vector
        sparse_query_vector = result.sparse_vector

    # Step 1: Determine starting directories
    if target_dirs:
        root_uris = target_dirs
    else:
        root_uris = self._get_root_uris_for_type(query.context_type, ctx=ctx)

    # Step 2: Global vector search
    global_results = await self._global_vector_search(
        vector_proxy=vector_proxy, query_vector=query_vector,
        sparse_query_vector=sparse_query_vector, context_type=context_type,
        target_dirs=target_dirs, scope_dsl=scope_dsl, limit=max(limit, self.GLOBAL_SEARCH_TOPK))

    # Step 3: Merge starting points
    starting_points = self._merge_starting_points(query.query, root_uris, global_results, mode=mode)

    # Step 4: Recursive search
    candidates = await self._recursive_search(vector_proxy=vector_proxy, query=query.query,
        query_vector=query_vector, sparse_query_vector=sparse_query_vector,
        starting_points=starting_points, limit=limit, mode=mode, threshold=effective_threshold, ...)

    # Step 6: Convert results
    matched = await self._convert_to_matched_contexts(candidates, ctx=ctx)
    final = matched[:limit]
```

### Root Directory Mapping (lines 589-618)

```python
def _get_root_uris_for_type(self, context_type: Optional[ContextType], ctx: Optional[RequestContext] = None) -> List[str]:
    if not ctx or ctx.role == Role.ROOT:
        return []

    user_space = ctx.user.user_space_name()
    agent_space = ctx.user.agent_space_name()

    if context_type is None:
        return [
            f"viking://user/{user_space}/memories",
            f"viking://agent/{agent_space}/memories",
            "viking://resources",
            f"viking://agent/{agent_space}/skills",
        ]
    elif context_type == ContextType.MEMORY:
        return [f"viking://user/{user_space}/memories", f"viking://agent/{agent_space}/memories"]
    elif context_type == ContextType.RESOURCE:
        return ["viking://resources"]
    elif context_type == ContextType.SKILL:
        return [f"viking://agent/{agent_space}/skills"]
    return []
```

### Recursive Search Algorithm (lines 351-498)

```python
async def _recursive_search(self, vector_proxy: VikingDBManagerProxy, query: str,
                            starting_points: List[Tuple[str, float]], limit: int, ...):
    collected_by_uri: Dict[str, Dict[str, Any]] = {}
    dir_queue: List[tuple] = []  # Priority queue: (-score, uri)
    visited: set = set()
    prev_topk_uris: set = set()
    convergence_rounds = 0

    # Initialize with starting points
    for uri, score in starting_points:
        heapq.heappush(dir_queue, (-score, uri))

    alpha = self.SCORE_PROPAGATION_ALPHA  # 0.5

    while dir_queue:
        temp_score, current_uri = heapq.heappop(dir_queue)
        current_score = -temp_score
        if current_uri in visited:
            continue
        visited.add(current_uri)

        # Search children of current directory
        results = await vector_proxy.search_children_in_tenant(
            parent_uri=current_uri, query_vector=query_vector,
            sparse_query_vector=sparse_query_vector, limit=pre_filter_limit)

        # Rerank if enabled
        if self._rerank_client and mode == RetrieverMode.THINKING:
            documents = [str(r.get("abstract", "")) for r in results]
            query_scores = self._rerank_scores(query, documents, query_scores)

        for r, score in zip(results, query_scores):
            uri = r.get("uri", "")
            # Score propagation: alpha * child + (1-alpha) * parent
            final_score = alpha * score + (1 - alpha) * current_score if current_score else score

            if not passes_threshold(final_score):
                continue

            # Deduplicate by URI, keep highest score
            previous = collected_by_uri.get(uri)
            if previous is None or final_score > previous.get("_final_score", 0):
                r["_final_score"] = final_score
                collected_by_uri[uri] = r

            # Only recurse into directories (L0/L1). L2 files are terminal.
            if uri not in visited and r.get("level", 2) != 2:
                heapq.heappush(dir_queue, (-final_score, uri))

        # Convergence check
        current_topk = sorted(collected_by_uri.values(), key=lambda x: x.get("_final_score", 0), reverse=True)[:limit]
        current_topk_uris = {c.get("uri", "") for c in current_topk}

        if current_topk_uris == prev_topk_uris and len(current_topk_uris) >= limit:
            convergence_rounds += 1
            if convergence_rounds >= self.MAX_CONVERGENCE_ROUNDS:
                break
        else:
            convergence_rounds = 0
            prev_topk_uris = current_topk_uris
```

### Score Propagation Formula

```
final_score = alpha * embedding_score + (1 - alpha) * parent_score
```

Where `alpha = 0.5` (SCORE_PROPAGATION_ALPHA)

This means child scores inherit 50% from their parent directory's score, ensuring that high-scoring directories boost all their children.

### Convergence Detection

The algorithm stops when:
1. The top-k URIs remain unchanged for `MAX_CONVERGENCE_ROUNDS` (3) consecutive iterations
2. AND we have at least `limit` candidates

This prevents infinite recursion while ensuring thorough exploration.

### Reranking Integration (lines 249-279)

```python
def _rerank_scores(self, query: str, documents: List[str], fallback_scores: List[float]) -> List[float]:
    if not self._rerank_client or not documents:
        return fallback_scores

    try:
        scores = self._rerank_client.rerank_batch(query, documents)
    except Exception as e:
        logger.warning("[HierarchicalRetriever] Rerank failed, fallback to vector scores: %s", e)
        return fallback_scores

    if not scores or len(scores) != len(documents):
        logger.warning("[HierarchicalRetriever] Invalid rerank result, fallback to vector scores")
        return fallback_scores
```

### Hotness Boost (lines 533-552)

After semantic scoring, a hotness factor can boost recently updated content:

```python
h_score = hotness_score(active_count=c.get("active_count", 0), updated_at=updated_at_val)
alpha = self.HOTNESS_ALPHA  # 0.2
final_score = (1 - alpha) * semantic_score + alpha * h_score
```

### Intent Analysis (from `openviking/retrieve/intent_analyzer.py`)

When using `search()` (vs `find()`), an LLM analyzes session context to generate multiple queries:

```python
class IntentAnalyzer:
    async def analyze(self, compression_summary: str, messages: List[Message],
                     current_message: Optional[str] = None, context_type: Optional[ContextType] = None,
                     target_abstract: str = "") -> QueryPlan:
        # Build context prompt with session history
        prompt = self._build_context_prompt(compression_summary, messages, current_message, context_type, target_abstract)
        # Call LLM
        response = await get_openviking_config().vlm.get_completion_async(prompt)
        # Parse result into TypedQuery list
```

### Query Result Types

**TypedQuery (input):**
```python
@dataclass
class TypedQuery:
    query: str              # Rewritten query
    context_type: ContextType  # MEMORY/RESOURCE/SKILL
    intent: str             # Query purpose
    priority: int           # 1-5 priority
    target_directories: Optional[List[str]] = None
```

**QueryResult (output):**
```python
@dataclass
class QueryResult:
    query: TypedQuery
    matched_contexts: List[MatchedContext]
    searched_directories: List[str]
```

**MatchedContext:**
```python
@dataclass
class MatchedContext:
    uri: str
    context_type: ContextType
    level: int  # 0, 1, or 2
    abstract: str
    category: str
    score: float
    relations: List[RelatedContext]
```

### find() vs search()

| Feature | find() | search() |
|---------|--------|---------|
| Session context | Not needed | Required |
| Intent analysis | Not used | LLM analysis |
| Query count | Single query | 0-5 TypedQueries |
| Latency | Low | Higher |
| Use case | Simple queries | Complex tasks |

### Clever Solutions

1. **Score propagation**: Child scores inherit parent context, forming semantic clusters
2. **Convergence detection**: Prevents infinite recursion while ensuring thorough search
3. **Level-based termination**: L2 files are terminal nodes; only directories continue recursion
4. **Hotness boost**: Recently/actively accessed content ranked higher
5. **Sparse vector support**: Hybrid dense+sparse embedding search

### Technical Debt / Concerns

1. **No grep_patterns implementation visible**: Comment mentions `grep_patterns` but implementation not clear in code
2. **Fallback rerank behavior**: Silently falls back to vector scores on rerank failure - may hide issues
3. **Priority queue tie-breaking**: Uses negative score for min-heap - works but non-obvious
4. **Initial candidates from global search**: L2 files added as initial candidates before recursion (lines 177-184)

---

## Cross-Feature Integration

### Feature Dependency Graph

```
VikingFS (viking:// Protocol)
    ├── Provides URI abstraction
    ├── Provides L0/L1/L2 file access
    ├── Provides relation management
    └── Used by HierarchicalRetriever for directory traversal

Tiered Context Loading
    ├── L0/L1 files stored via VikingFS
    ├── Session compression creates L0/L1
    └── HierarchicalRetriever reads L0/L1 for scoring

HierarchicalRetriever
    ├── Uses VikingFS for relations and abstract reading
    ├── Uses VectorDB for semantic search
    └── Integrates with IntentAnalyzer for session-aware search
```

### Data Flow Example: Adding a Resource

1. Resource content arrives via API
2. Parser extracts structured content
3. SemanticProcessor generates L0/L1 bottom-up:
   - Leaf files: L0 = first ~100 tokens, L1 = structured summary
   - Directories: L0 aggregated from children, L1 = navigation overview
4. Content stored via VikingFS `write()`:
   - L2 content: `{uri}/content.md`
   - L0: `{uri}/.abstract.md`
   - L1: `{uri}/.overview.md`
5. Vector embeddings created from L0 text
6. VectorDB updated with URI, level, parent_uri

### Data Flow Example: Retrieval

1. User issues query via `find()` or `search()`
2. If `search()`: IntentAnalyzer generates TypedQueries from session context
3. HierarchicalRetriever:
   - Global vector search finds initial directory candidates
   - Priority queue explores directories recursively
   - Scores propagate from parent to children
   - Convergence detected after stable top-k
4. Results include URI, level, abstract, relations
5. Client can read L2 content on-demand via VikingFS

---

## File Locations Summary

| Component | File Path |
|-----------|-----------|
| VikingFS | `openviking/storage/viking_fs.py` |
| URI Utilities | `openviking_cli/utils/uri.py` |
| Hierarchical Retriever | `openviking/retrieve/hierarchical_retriever.py` |
| Intent Analyzer | `openviking/retrieve/intent_analyzer.py` |
| Session Compressor | `openviking/session/compressor.py` |
| Viking URI Docs | `docs/en/concepts/04-viking-uri.md` |
| Context Layers Docs | `docs/en/concepts/03-context-layers.md` |
| Retrieval Docs | `docs/en/concepts/07-retrieval.md` |

---

## Key Implementation Patterns Observed

1. **Singleton pattern** for VikingFS (`init_viking_fs()`, `get_viking_fs()`)
2. **Context variable binding** for request context propagation
3. **Proxy wrapper** pattern for tenant-safe operations (`VikingDBManagerProxy`)
4. **Priority queue with score propagation** for hierarchical traversal
5. **Graceful degradation** (rerank failures fall back to vector scores)
6. **Idempotent operations** (rm succeeds on non-existent paths)
7. **Bottom-up processing** for L0/L1 generation
8. **Chunking with overlap** for long content indexing
