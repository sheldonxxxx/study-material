# Feature Batch 4 Analysis (Features 10-12)

## Feature 10: Workspace Boundary & Privacy

### Description
Privacy controls preventing cross-workspace memory contamination, with configurable boundary rules for user profile/name/addressing data.

### Key Implementation Files

| File | Size | Purpose |
|------|------|---------|
| `src/workspace-boundary.ts` | 155 lines | Core boundary enforcement logic |
| `src/admission-control.ts` | 749 lines | Memory admission scoring system |
| `src/identity-addressing.ts` | 202 lines | Name/addressing extraction patterns |

---

### Core Logic Flow

#### 1. Workspace Boundary Configuration (`workspace-boundary.ts`)

**Configuration Resolution** (lines 44-56):
```typescript
resolveUserMdExclusiveConfig(workspaceBoundary?)
  -> Returns ResolvedUserMdExclusiveConfig
  -> Keys: enabled, routeProfile, routeCanonicalName, routeCanonicalAddressing, filterRecall
```

**Memory Classification** (lines 64-125):
1. `isUserMdExclusiveMemory()` checks if a memory entry belongs to exclusive slots
2. Uses `classifyIdentityAndAddressingMemory()` from `identity-addressing.ts` to detect `name` and `addressing` slots
3. Applies `PROFILE_HINT_PATTERNS` regex for profile detection
4. Returns `true` if memory matches any enabled exclusive slot

**Filtering** (lines 145-154):
- `filterUserMdExclusiveRecallResults()` filters out user-md-exclusive memories from recall results when `filterRecall` is enabled

#### 2. Identity & Addressing Detection (`identity-addressing.ts`)

**Pattern-Based Extraction**:
- `NAME_PATTERNS` (lines 45-49): Extracts Chinese/English names
  - Chinese: `/(?:我的名字是|我(?:现在)?叫|本名是)\s*([^\s，。,.!！?？"'""''「」『』]+)/iu`
  - English: `/name\s+is\s+['"]?([^'".,\n]+)['"]?/i`
- `ADDRESSING_PATTERNS` (lines 51-56): Extracts preferred address/alias

**Slot Classification** (lines 127-165):
- `classifyIdentityAndAddressingMemory()` returns a `Set<IdentityAddressingSlot>` containing `"name"` and/or `"addressing"`
- Canonical fact keys `CANONICAL_NAME_FACT_KEY` and `CANONICAL_ADDRESSING_FACT_KEY` are checked

**Candidate Creation** (lines 91-112):
- `createIdentityAndAddressingCandidates()` extracts name/addressing from text and creates `CandidateMemory` objects
- Canonicalizes extracted values to proper categories (`entities` for name, `preferences` for addressing)

---

#### 3. Admission Control System (`admission-control.ts`)

**Purpose**: Decides whether a candidate memory should be admitted to storage based on weighted scoring.

**Scoring Components** (lines 106-121):

| Component | Weight (balanced) | Description |
|-----------|------------------|-------------|
| `typePrior` | 0.6 | Category-specific prior probability |
| `utility` | 0.1 | LLM-evaluated future usefulness |
| `confidence` | 0.1 | Support from conversation context |
| `novelty` | 0.1 | Dissimilarity to existing memories |
| `recency` | 0.1 | Gap since last similar memory |

**Scoring Algorithms**:

1. **Type Prior** (lines 508-513): Direct lookup from `AdmissionTypePriors`
   ```typescript
   profile: 0.95, preferences: 0.9, entities: 0.75, cases: 0.8, patterns: 0.85, events: 0.45
   ```

2. **Confidence Support** (lines 515-541): ROUGE-Like F1 between candidate and conversation spans
   ```typescript
   score = clamp01((bestSupport * 0.7) + (coverage * 0.3) - (unsupportedRatio * 0.25), 0)
   ```

3. **Novelty** (lines 543-572): Cosine similarity to existing memories
   ```typescript
   score = clamp01(1 - maxSimilarity, 1)
   ```

4. **Recency Gap** (lines 574-598): Exponential decay based on gap
   ```typescript
   score = clamp01(1 - Math.exp(-lambda * gapDays), 1)
   ```

5. **Utility** (lines 600-628): LLM completion call scoring future usefulness

**Decision Logic** (lines 666-747):
```typescript
score = weightedSum(featureScores)
decision = score < rejectThreshold ? "reject" : "pass_to_dedup"
hint = score >= admitThreshold && novelty < 0.55 ? "add" : "update_or_merge"
```

**Presets** (lines 132-207):
- `balanced`: Default, moderate thresholds
- `conservative`: Higher thresholds, favors existing memories
- `high-recall`: Lower thresholds, admits more memories

---

### Notable Patterns

1. **Separation of Concerns**: `workspace-boundary.ts` handles filtering only; actual scoring is in `admission-control.ts`

2. **Fail-Open Design**: `resolveUserMdExclusiveConfig()` defaults to `enabled: false`, preventing accidental data loss

3. **Audit Trail**: Full admission decisions are logged with all feature scores, thresholds, and matched memory IDs

4. **Tokenization for CJK**: `tokenizeText()` (lines 361-392) handles Han characters as individual tokens (not splitting), preserving CJK semantics

5. **LCS-based Similarity**: `rougeLikeF1()` uses Longest Common Subsequence for confidence scoring (not simple n-gram overlap)

---

### Technical Debt & Concerns

1. **Admission Controller is sync but utility scoring is async**: The `AdmissionController.evaluate()` is async but `scoreUtility()` is the only async component. No parallel execution within evaluate.

2. **Hardcoded 0.55 novelty threshold**: Magic number at line 561, 710 - should be configurable.

3. **CJK character handling**: `isHanChar()` uses `/p{Script=Han}/u` which may not work in older Node versions without full Unicode property support.

4. **No dedup step in admission**: Comments mention "pass_to_dedup" but actual deduplication happens post-admission in the storage pipeline.

---

## Feature 11: Reflection System

### Description
Learning from agent behavior patterns with reflection slices, event stores, and governance metadata for self-improvement. Enables the agent to reflect on its own behavior and store insights that influence future runs.

### Key Implementation Files

| File | Size | Purpose |
|------|------|---------|
| `src/reflection-store.ts` | 605 lines | Main reflection storage/loading logic |
| `src/reflection-slices.ts` | 319 lines | Markdown parsing & slice extraction |
| `src/reflection-event-store.ts` | 98 lines | Event ID generation & payload building |
| `src/reflection-item-store.ts` | 112 lines | Individual item metadata/payloads |
| `src/reflection-mapped-metadata.ts` | 84 lines | Mapped memory metadata |
| `src/reflection-ranking.ts` | 33 lines | Logistic score computation |
| `src/reflection-metadata.ts` | 23 lines | Metadata parsing utilities |
| `src/intent-analyzer.ts` | 260 lines | Adaptive recall intent routing |

---

### Core Logic Flow

#### 1. Reflection Storage Pipeline

**Entry Point**: `storeReflectionToLanceDB()` (`reflection-store.ts`, lines 185-217)

**Step 1: Build Payloads** (`buildReflectionStorePayloads()`, lines 56-114)
```typescript
reflectionText -> extractInjectableReflectionSlices()
                          |
                          v
                  ReflectionSlices { invariants: string[], derived: string[] }
                          |
                          v
     +-------------------+-------------------+
     |                   |                   |
     v                   v                   v
buildReflectionEvent  buildReflectionItem  buildLegacyCombined
   Payload               Payload             Payload
     |                   |                   |
     v                   v                   v
"event" kind       "item-invariant"    "combined-legacy"
                       "item-derived"
```

**Step 2: Deduplicate** (lines 195-203)
- Vector search against existing memories in scope
- Skip if similarity > 0.97 threshold

**Step 3: Embed & Store** (lines 205-213)
- Each payload embedded and stored separately
- Importance resolved by kind: event=0.55, invariant=0.82, derived=0.78, legacy=0.75

---

#### 2. Reflection Slice Extraction (`reflection-slices.ts`)

**Source Text Parsing** (lines 37-54):
```typescript
extractSectionMarkdown(markdown, heading)
  -> Finds "## {heading}" section
  -> Collects lines until next "## " or EOF
```

**Bullet Extraction** (lines 56-67):
```typescript
parseSectionBullets(markdown, heading)
  -> extractSectionMarkdown()
  -> Filter lines starting with "- " or "* "
```

**Sanitization Pipeline**:
1. `normalizeReflectionSliceLine()`: Removes `**`, section prefixes
2. `isPlaceholderReflectionSliceLine()`: Filters placeholder text
3. `isUnsafeInjectableReflectionLine()`: Filters prompt injection attempts

**Injection Prevention** (lines 93-106):
```typescript
INJECTABLE_REFLECTION_BLOCK_PATTERNS = [
  /^\s*(?:ignore|disregard|forget|override|bypass)\b.../i,  // Override attempts
  /\b(?:reveal|print|dump|show)\b...\b(system prompt|secrets?)\b/i,  // Extraction attempts
  /<\s*\/?\s*(?:system|assistant|user|tool|...)\b[^>]*>/i,  // XML tag injection
  /^(?:system|assistant|user|developer|tool)\s*:/i,  // Role play prefixes
]
```

**Slice Classification** (lines 114-122):
- `isInvariantRuleLike()`: Detects rule statements (`always`, `never`, `must`, `should`)
- `isDerivedDeltaLike()`: Detects change-oriented statements (`this run`, `next run`, `adjust`)

---

#### 3. Reflection Ranking & Scoring (`reflection-ranking.ts`)

**Logistic Decay Model** (lines 12-17):
```typescript
computeReflectionLogistic(ageDays, midpointDays, k)
  = 1 / (1 + exp(k * (ageDays - midpointDays)))
```

**Score Composition** (lines 19-25):
```typescript
score = logistic * baseWeight * quality * fallbackFactor
  fallbackFactor = 0.75 if usedFallback else 1.0
```

**Decay Parameters by Type**:

| Type | Midpoint (days) | k | Base Weight |
|------|----------------|---|-------------|
| Invariant | 45 | 0.22 | 1.1 |
| Derived | 7 | 0.65 | 1.0 |
| Mapped: decision | 45 | 0.25 | 1.1 |
| Mapped: user-model | 21 | 0.3 | 1.0 |
| Mapped: agent-model | 10 | 0.35 | 0.95 |
| Mapped: lesson | 7 | 0.45 | 0.9 |

---

#### 4. Reflection Loading (`loadAgentReflectionSlicesFromEntries()`, lines 234-271)

**Step 1: Filter & Sort** (lines 246-250)
- Parse metadata, filter by type (`memory-reflection-item` or `memory-reflection`)
- Filter by agent ownership
- Sort by timestamp descending, take top 160

**Step 2: Build Candidates** (lines 255-256)
- Invariant candidates from item rows, fallback to legacy
- Derived candidates same pattern

**Step 3: Rank & Limit** (lines 258-268)
- `rankReflectionLines()` aggregates scores across sessions
- Normalizes lines for deduplication (lowercase, whitespace collapse)
- Returns top 8 invariants, top 10 derived

---

#### 5. Mapped Memory System (`loadReflectionMappedRowsFromEntries()`, lines 492-583)

**Categories**:
- `user-model`: Deltas about the human
- `agent-model`: Deltas about the assistant
- `lesson`: Pitfalls with symptom/cause/fix/prevention
- `decision`: Durable decisions

**Key Insight**: Unlike simple slices, mapped memories maintain their `mappedKind` category and are scored/ranked separately per kind (line 567-575).

---

#### 6. Intent Analyzer for Adaptive Recall (`intent-analyzer.ts`)

**Purpose**: Determines which memory categories to prioritize based on query patterns, minimizing token cost for auto-recall.

**Intent Rules** (lines 58-122):

| Intent | Patterns | Categories | Depth |
|--------|----------|------------|-------|
| preference | `prefer`, `like`, `dislike`, `style` | preference, decision | l0 |
| decision | `why did`, `decision`, `chose`, `rationale` | decision, fact | l1 |
| entity | `who is`, `tell me about`, `contact info` | entity, fact | l1 |
| event | `when did`, `what happened`, `timeline` | entity, decision | full |
| fact | `how does`, `what is`, `explain`, `spec` | fact, entity | l1 |

**Category Boosting** (lines 179-196):
```typescript
applyCategoryBoost(results, intent, boostFactor=1.15)
  -> For matching categories: score *= boostFactor (capped at 1.0)
  -> Re-sorts results
```

**Formatting by Depth** (lines 205-242):
- `l0`: First sentence or 80 chars
- `l1`: Up to 300 chars
- `full`: Complete text

---

### Notable Patterns

1. **Three Storage Formats**: The system maintains backward compatibility with legacy `combined-legacy` format while using separate `event` and `item-*` entries for granular scoring.

2. **Normalization for Deduplication**: Lines are normalized (lowercase, whitespace collapsed) before aggregation, preventing duplicates from phrasing variations.

3. **Safety-First Injection Prevention**: Extensive regex patterns block prompt injection via reflection slices - a critical security concern for a self-modifying system.

4. **Adaptive Recall Without LLM**: Intent analysis is pure rule-based (no LLM call), ensuring minimal latency for auto-recall decisions.

5. **Hierarchical Fallback**: Reflection loading falls back through: item rows -> legacy rows -> empty.

6. **Quality Scoring Based on Volume**: `computeDerivedLineQuality()` assigns higher quality to reflections with more bullets (0.55 + n*0.075, capped at 1.0).

---

### Technical Debt & Concerns

1. **Reflection Schema Version 4**: The system has undergone 3 prior revisions (`reflectionVersion: 4`). Migration path for older entries is unclear.

2. **Max Age Per Type**: Invariants and derived have different max ages (14 days vs 60 days), but these are not enforced during ranking - only during loading.

3. **Line 273-274 Bug**: The fallback logic for derived uses `reflectionLinesLegacy` filtered with `isDerivedDeltaLike` but this pattern may not match legacy format content properly.

4. **No Source Validation**: `sourceReflectionPath` is accepted but never validated against actual file existence.

5. **Intent Analyzer Bilingual**: Rules include Chinese patterns but the bilingual approach may miss edge cases or become unwieldy as patterns grow.

---

## Feature 12: Long-Context Chunking

### Description
Intelligent chunking strategy for processing long documents with configurable overlap and size limits, triggered when embedding providers throw context-length errors.

### Key Implementation Files

| File | Size | Purpose |
|------|------|---------|
| `src/chunker.ts` | 285 lines | Core chunking algorithms |
| `docs/long-context-chunking.md` | 259 lines | Documentation |

---

### Core Logic Flow

#### 1. Public API (`chunkDocument()`, lines 195-256)

**Flow**:
```
while pos < text.length:
  1. If remaining <= maxChunkSize: Take remainder as final chunk
  2. Otherwise:
     a. findSplitEnd() at semantic boundary (sentence > newline > whitespace)
     b. sliceTrimWithIndices() to extract and trim chunk
     c. If trimmed chunk < minChunkSize: Fall back to hard split
     d. Push chunk
     e. pos = end - overlapSize (move forward with overlap)
```

**Guardrails**:
- `maxGuard`: Prevents infinite loops (4 + text.length/optimalChunkSize + 5)
- Hard fallback if trimming destroys semantic coherence

---

#### 2. Smart Chunking (`smartChunk()`, lines 264-282)

**Context Limit Resolution**:
```typescript
limit = EMBEDDING_CONTEXT_LIMITS[embedderModel] ?? 8192
```

**CJK Adaptation** (lines 270-271):
```typescript
cjkRatio = getCjkRatio(text)  // CJK chars / total non-whitespace chars
if cjkRatio > 0.3:
  divisor = 2.5  // CJK chars are ~2-3 tokens each
else:
  divisor = 1
```

**Conservative Sizing**:
```typescript
maxChunkSize = floor(base * 0.7 / divisor)   // 70% of limit
overlapSize = floor(base * 0.05 / divisor)  // 5% of limit
minChunkSize = floor(base * 0.1 / divisor)  // 10% of limit
```

---

#### 3. Split Point Finding (`findSplitEnd()`, lines 97-144)

**Priority Order**:
1. **Line limit enforcement**: If `maxLinesPerChunk` exceeded, split at Nth newline
2. **Semantic split**: Scan backward for sentence-ending punctuation
3. **Newline boundary**: Fall back to line break
4. **Whitespace boundary**: Last resort word break

**Sentence Endings** (line 72):
```typescript
const SENTENCE_ENDING = /[.!?。！？]/
```

**CJK Handling**: Sentence scanning includes `。！？` in addition to English punctuation.

---

#### 4. Slice Trimming (`sliceTrimWithIndices()`, lines 146-163)

**Problem**: Raw chunk boundaries often include leading/trailing whitespace.

**Solution**:
```typescript
leading = raw.match(/^\s*/)?.[0]?.length ?? 0
trailing = raw.match(/\s*$/)?.[0]?.length ?? 0
chunk = raw.trim()

meta = {
  startIndex: start + leading,
  endIndex: end - trailing,
  length: chunk.length
}
```

**Key Insight**: Metadata indices point to trimmed content positions, not raw slice positions.

---

#### 5. CJK Detection (`getCjkRatio()`, lines 175-184)

**Unicode Ranges** (line 171-172):
```typescript
const CJK_RE = /[\u3040-\u309F\u30A0-\u30FF\u3400-\u4DBF\u4E00-\u9FFF\uAC00-\uD7AF\uF900-\uFAFF]/
```

**Ratio Calculation**:
```typescript
cjkChars / nonWhitespaceChars
```

Note: Uses character count as proxy for token count (as stated in file header line 8).

---

### Supported Embedding Models

| Model | Context Limit | Chunk Size (70%) | Overlap (5%) |
|-------|--------------|------------------|--------------|
| jina-embeddings-v5-text-small | 8192 | 5734 | 409 |
| text-embedding-3-small | 8192 | 5734 | 409 |
| text-embedding-3-large | 8192 | 5734 | 409 |
| gemini-embedding-001 | 2048 | 1433 | 102 |
| nomic-embed-text | 8192 | 5734 | 409 |
| all-MiniLM-L6-v2 | 512 | 358 | 40 |
| all-mpnet-base-v2 | 512 | 358 | 40 |

---

### Notable Patterns

1. **Reactive Not Preventive**: The chunker is triggered by provider errors, not pre-emptive analysis. This is stated in the file header: "The embedder triggers this only after a provider throws a context-length error."

2. **Conservative Character-Based Limits**: Using 70% of reported token limit as character limit provides safety margin since char/token ratio varies.

3. **Overlap Strategy**: Overlap at `end - overlapSize` means chunks share boundary content, preserving cross-chunk context.

4. **Hard Fallback on Trim Failure**: If semantic splitting produces a chunk < `minChunkSize`, a hard split at `maxChunkSize` is used instead.

5. **Empty Document Guard**: Early return for empty/whitespace-only input (line 196-198).

---

### Technical Debt & Concerns

1. **No Document-Level Embedding Averaging in Source**: The `chunkDocument()` function returns chunks, but the actual embedding averaging happens in the embedder (not visible in this file). The docs show this flow but source implementation is elsewhere.

2. **Character Count as Token Proxy**: The header explicitly admits this is a "conservative proxy" (line 8). CJK ratio helps but doesn't fully solve the problem.

3. **Hardcoded Magic Numbers**: `0.7`, `0.05`, `0.1`, `0.3`, `2.5` are scattered without named constants explaining rationale.

4. **No Hierarchical Chunking**: Docs mention this as a planned enhancement (line 218). Current approach averages embeddings, losing document-level structure.

5. **Guard Loop Could Get Stuck**: If `nextPos` calculation results in very small advances (e.g., overlapSize ~ maxChunkSize), the loop might produce many tiny chunks before completing.

6. **Single Document Hash for Caching**: If the same long document is stored multiple times with minor edits, each will recompute chunks separately. No sub-document caching strategy.

---

### Edge Cases Handled

| Case | Handling |
|------|----------|
| Empty document | Returns `{chunks: [], metadatas: [], totalOriginalLength: 0, chunkCount: 0}` |
| Document <= maxChunkSize | Single chunk, no splitting |
| No sentence boundaries | Falls back to newline, then whitespace |
| All whitespace | Trim results in empty chunk, skipped |
| Single very long word | Hard split at maxChunkSize |
| CJK-only text | Ratio-based divisor adjusts limits |

---

## Cross-Cutting Observations

### Security Considerations

1. **Reflection Injection Prevention**: Feature 11 has extensive sanitization for prompt injection via reflection slices
2. **Workspace Boundary Filtering**: Feature 10 prevents cross-workspace data leakage at recall time

### Performance Patterns

1. **Intent Analyzer is Sync**: No LLM calls, pure regex matching
2. **Admission Control Scoring**: All components are synchronous except utility scoring (async LLM call)
3. **Chunking is CPU-Bound**: Pure string manipulation, no I/O

### Data Model Evolution

1. **Reflection Schema v4**: Multiple schema versions coexist; migration not visible
2. **Legacy Format Support**: Feature 11 maintains backward compatibility with older reflection formats
3. **Metadata Extensibility**: All systems use JSON metadata blobs with typed parsing on read

### Missing Testability Signals

1. **No visible test coverage** in these files
2. **Magic numbers without explanation** (especially in chunker)
3. **Error handling is implicit** - failures often result in silent fallthrough rather than explicit error propagation
