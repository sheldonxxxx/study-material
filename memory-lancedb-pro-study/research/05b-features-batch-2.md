# Feature Batch 2 Deep Dive: Features 4-6

## Feature 4: Smart Memory Extraction

### Description
LLM-powered extraction into 6 semantic categories (profile, preferences, entities, events, cases, patterns) with L0/L1/L2 layered storage and two-stage deduplication.

### Key Implementation Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/smart-extractor.ts` | 1372 | Main extraction pipeline, SmartExtractor class |
| `src/extraction-prompts.ts` | 217 | LLM prompt templates (extract, dedup, merge) |
| `src/smart-metadata.ts` | 665 | Metadata handling, fact keys, support info V1/V2 |
| `src/memory-categories.ts` | 87 | Category definitions and normalization |
| `src/batch-dedup.ts` | 147 | Cosine similarity dedup within extraction batches |

### Core Logic Flow

```
extractAndPersist(conversationText)
  │
  ├─► Step 1: extractCandidates()
  │     ├─ Truncate text to extractMaxChars (default 8000)
  │     ├─ stripEnvelopeMetadata() — remove OpenClaw channel metadata
  │     ├─ buildExtractionPrompt(cleaned, user) → LLM
  │     └─ Validate: normalizeCategory, isNoise, min abstract length
  │
  ├─► Step 1b: batchDedup() — BEFORE expensive LLM dedup
  │     ├─ Embed candidate abstracts in parallel
  │     ├─ Pairwise cosine similarity O(n²), n ≤ 5
  │     └─ Drop candidates with similarity > 0.85
  │
  └─► Step 2: processCandidate() — for each survivor
        │
        ├─► ALWAYS_MERGE categories (profile) → handleProfileMerge()
        │
        ├─► Vector search for similar existing memories
        │
        ├─► Admission control gate (before dedup)
        │     └─ evaluate() → reject/create/hold
        │
        └─► deduplicate() → LLM decision
              ├─ "skip" — duplicate, no action
              ├─ "create" — store as new
              ├─ "merge" — LLM merge with existing (preferences/entities/patterns)
              ├─ "supersede" — create new, invalidate old (temporal categories)
              ├─ "support" — update context support stats
              ├─ "contextualize" — create linked contextual variant
              └─ "contradict" — record contradiction evidence
```

### Three-Level Memory Structure

Each extracted memory has three levels stored in metadata:

| Level | Field | Purpose |
|-------|-------|---------|
| L0 | `l0_abstract` | One-line index used for search/BM25 text |
| L1 | `l1_overview` | Structured Markdown summary with category-specific headings |
| L2 | `l2_content` | Full narrative with background and details |

### Six Memory Categories

| Category | Dedup Strategy | Description |
|----------|---------------|-------------|
| `profile` | ALWAYS_MERGE | User identity (name, occupation, background) |
| `preferences` | MERGE/SUPERSEDE | User preferences and habits |
| `entities` | MERGE/SUPERSEDE | People, projects, organizations |
| `events` | APPEND_ONLY | Decisions, milestones, completed actions |
| `cases` | APPEND_ONLY | Problem-solution pairs |
| `patterns` | MERGE | Reusable processes and procedures |

### Notable Patterns and Techniques

**1. Two-Stage Deduplication**
- Stage 1: Cosine similarity pre-filter (cheap, fast) on candidate abstracts
- Stage 2: LLM decision (expensive) only for survivors
- This dramatically reduces LLM API calls

**2. Envelope Metadata Stripping (lines 72-97)**
```typescript
// Strips OpenClaw channel metadata that would pollute extraction:
// - "System: [timestamp] Channel[account] ..." lines
// - "Conversation info (untrusted metadata):" + JSON blocks
// - "Sender (untrusted metadata):" + JSON blocks
```
Weaker LLMs (e.g., qwen) store metadata verbatim, so it must be removed before extraction.

**3. Preference Slot Guard (lines 648-666)**
Same brand but different item-level preferences are NOT merged:
```typescript
// "喜欢麦当劳的板烧鸡腿堡" vs "喜欢麦当劳的麦辣鸡翅"
// These are different items even if same brand — stored separately
```

**4. Fact Key for Temporal Versioning (smart-metadata.ts:202-227)**
Fact keys enable supersession chains for temporal facts:
```typescript
// Extracts merge key from abstract using colon/arrow delimiters
// "Preferred editor: VS Code" → topic "Preferred editor"
// Used to link superseded/superseded_by memories
```

**5. Contextual Support Stats (V2)**
Support info tracks confirmations/contradictions per context:
```typescript
// Contexts: general, morning, evening, night, weekday, weekend, work, leisure, summer, winter, travel
// Slices capped at MAX_SUPPORT_SLICES = 8 to prevent metadata bloat
```

**6. Admission Control Integration**
Governance layer gates memory persistence before dedup:
- Evaluates candidate quality/relevance
- Rejects with audit trail if configured
- Rejections tracked in `ExtractionStats.rejected`

### Technical Debt and Concerns

1. **Profile always merges without dedup** — The profile category skips the full dedup pipeline. If the LLM extracts conflicting profile info, it gets merged unconditionally. May need a conflict resolution strategy.

2. **No rollback on partial failures** — If `processCandidate` throws after some memories are stored but before others complete, the batch is left partially processed.

3. **LLM Merge produces intermediate artifacts** — The merge prompt asks LLM to produce merged L0/L1/L2, but there's no validation the output is better than either input.

4. **Hardcoded similarity thresholds** — `SIMILARITY_THRESHOLD = 0.7` for vector search and `0.85` for batch dedup are not configurable without code changes.

5. **No extraction timeout** — LLM calls have no timeout. A misbehaving LLM could cause extraction to hang indefinitely.

6. **Support slice truncation drift** — When slices are truncated to MAX_SUPPORT_SLICES, evidence from dropped slices is partially preserved in global_strength but the per-slice counts are lost. The code acknowledges this trade-off.

---

## Feature 5: Memory Lifecycle Management

### Description
Weibull decay engine with composite scoring (recency + frequency + importance) and three-tier promotion system (Peripheral, Working, Core).

### Key Implementation Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/decay-engine.ts` | 229 | Weibull decay model, composite scoring |
| `src/tier-manager.ts` | 189 | Tier promotion/demotion logic |
| `src/access-tracker.ts` | 341 | Access frequency tracking with debounced write-back |
| `src/smart-metadata.ts` | 447-498 | toLifecycleMemory, getDecayableFromEntry adapters |

### Core Decay Algorithm

**Composite Score = 0.4 × recency + 0.3 × frequency + 0.3 × intrinsic**

**Recency (Weibull stretched-exponential):**
```typescript
effectiveHL = halfLife * exp(1.5 * importance)
lambda = ln(2) / effectiveHL
recency = exp(-lambda * daysSince^beta)

// beta varies by tier:
// Core:      beta = 0.8  (sub-exponential, slower decay)
// Working:   beta = 1.0  (standard exponential)
// Peripheral: beta = 1.3  (super-exponential, faster decay)
```

**Frequency (logarithmic saturation):**
```typescript
base = 1 - exp(-accessCount / 5)
recentBonus = exp(-avgGapDays / 30)  // only if accessCount > 1
frequency = base * (0.5 + 0.5 * recentBonus)
```

**Intrinsic:** `importance × confidence`

**Decay Floors (minimum composite by tier):**
- Core: 0.9 — identity facts almost never forgotten
- Working: 0.7 — active context, ages without reinforcement
- Peripheral: 0.5 — low priority or aging memories

### Tier Promotion/Demotion Rules

| Transition | Condition |
|------------|-----------|
| Peripheral → Working | accessCount ≥ 3 AND composite ≥ 0.4 |
| Working → Core | accessCount ≥ 10 AND composite ≥ 0.7 AND importance ≥ 0.8 |
| Working → Peripheral | composite < 0.15 OR (age > 60 days AND accessCount < 3) |
| Core → Working | composite < 0.15 AND accessCount < 3 |

### Access Tracker Design

The AccessTracker implements **debounced batch write-back**:

```
recordAccess(ids) → synchronous Map update only (no I/O)
                         │
                    debounce timer (default 5s)
                         │
                    flush() → read-modify-write each memory
```

**Key features:**
- Access count decays with 30-day half-life (access freshness)
- Effective half-life = `baseHL + baseHL * reinforcementFactor * log1p(effectiveAccessCount)`
- Hard cap on multiplier (default 5x)
- Flush failures requeue deltas for retry
- Both camelCase and snake_case written for compatibility

### Notable Patterns

**1. Importance-Modulated Half-Life**
Higher importance → longer effective half-life → slower decay:
```typescript
effectiveHL = halfLife * exp(1.5 * importance)
// importance 0.7 → effectiveHL ≈ 30 * 2.86 ≈ 86 days
// importance 0.9 → effectiveHL ≈ 30 * 3.67 ≈ 110 days
```

**2. Tier-Specific Weibull Beta**
Different decay curves per tier:
- Core (β=0.8): Stretched exponential — memory persists longer
- Working (β=1.0): Standard exponential
- Peripheral (β=1.3): Super-exponential — faster degradation

**3. Search Boost Application**
Decay scores applied as multiplier to search results:
```typescript
tierFloor = max(getTierFloor(tier), compositeScore)
multiplier = boostMin + (1 - boostMin) * tierFloor
// Core memories with high composite get up to 1.0x boost
// Peripheral memories with low composite get boostMin (0.3) floor
```

### Technical Debt and Concerns

1. **Time-dependent thresholds** — Age-based demotion uses calendar days, which means memories created at different times have different decay rates even with identical access patterns.

2. **Access count is a proxy metric** — Frequency is measured by access COUNT, not access QUALITY. A user accessing the same memory 10 times briefly counts the same as 10 thoughtful retrievals.

3. **No distributed decay coordination** — If multiple OpenClaw instances run simultaneously, they each maintain their own AccessTracker. Flush timing conflicts could cause lost increments.

4. **Demotion is one-way** — The Core → Working demotion threshold (composite < 0.15 AND accessCount < 3) is very aggressive. Core memories rarely demote back to Working.

5. **No explicit promotion budget** — There's no cap on how many memories can be Core or Working. With enough access, all memories could promote to Core.

---

## Feature 6: Auto-Capture & Auto-Recall

### Description
Automatic memory extraction on `agent_end` hook and context injection via `before_prompt_build` hook. Filters noise and adaptively skips retrieval for non-memory queries.

### Key Implementation Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/noise-filter.ts` | 97 | Pattern-based noise filtering |
| `src/adaptive-retrieval.ts` | 98 | Retrieval decision logic |
| `src/auto-capture-cleanup.ts` | 95 | Text normalization for auto-capture |

### Noise Filter (`noise-filter.ts`)

**Four noise categories:**

| Category | Patterns |
|----------|----------|
| Denials | "I don't have any information", "I don't recall", "no relevant memories found" |
| Meta-questions | "Do you remember X?", "Did I tell you about Y?", Chinese recall patterns |
| Boilerplate | "hi", "hello", "fresh session", "new session", "HEARTBEAT" |
| Diagnostic artifacts | "query -> none", "no explicit solution", "user asked for... none" |

**Filter logic:**
```typescript
function isNoise(text: string): boolean {
  if (trimmed.length < 5) return true;
  if (DENIAL_PATTERNS.some(p => p.test(trimmed))) return true;
  if (META_QUESTION_PATTERNS.some(p => p.test(trimmed))) return true;
  if (BOILERPLATE_PATTERNS.some(p => p.test(trimmed))) return true;
  if (DIAGNOSTIC_ARTIFACT_PATTERNS.some(p => p.test(trimmed))) return true;
  return false;
}
```

### Adaptive Retrieval (`adaptive-retrieval.ts`)

**Decision tree:**

```
shouldSkipRetrieval(query)
  │
  ├─► Check FORCE_RETRIEVE_PATTERNS first
  │     └─ If matched, ALWAYS retrieve (memory keywords, personal references)
  │
  ├─► Length check: < 5 chars → skip
  │
  ├─► SKIP_PATTERNS:
  │     ├─ Greetings (hi, hello, hey...)
  │     ├─ Slash commands (/)
  │     ├─ Simple affirmations (yes, no, ok, thanks...)
  │     ├─ Continuation prompts (go ahead, continue...)
  │     ├─ Pure emoji
  │     └─ Heartbeat/system messages
  │
  ├─► Language-aware length threshold
  │     ├─ CJK text: < 6 chars (without ? or ？) → skip
  │     └─ Non-CJK text: < 15 chars (without ? or ？) → skip
  │
  └─► Default: retrieve
```

**OpenClaw metadata stripping:**
```typescript
// Strips [cron:...] wrappers
// Strips "Conversation info (untrusted metadata):" JSON blocks
// Strips "[Mon 2026-03-02 04:21 GMT+8]" timestamp prefixes
```

### Auto-Capture Cleanup (`auto-capture-cleanup.ts`)

**Normalization pipeline:**

```
stripAutoCaptureInjectedPrefix(role, text)
  │
  ├─► Strip <relevant-memories>...</relevant-memories> blocks
  ├─► Strip [UNTRUSTED DATA...]...[END UNTRUSTED DATA] blocks
  ├─► Strip "A new session was started via /new or /reset..." prefix
  ├─► Strip inbound metadata blocks (System:, Sender:, Replied message...)
  ├─► Strip @mention addressing prefixes (@user )
  ├─► Strip system event lines (System: [...] Exec completed...)
  └─► Collapse excessive newlines (3+ → 2)
```

**Key sentinels stripped:**
- `Conversation info (untrusted metadata):`
- `Sender (untrusted metadata):`
- `Thread starter (untrusted, for context):`
- `Replied message (untrusted, for context):`
- `Forwarded message context (untrusted metadata):`
- `Chat history since last reply (untrusted, for context):`

### Notable Patterns

**1. CJK-Aware Length Threshold**
Chinese/Japanese/Korean text uses shorter minimum length because characters carry more semantic density:
```typescript
const hasCJK = /[\u4e00-\u9fff\u3040-\u309f\u30a0-\u30ff\uac00-\ud7af]/.test(trimmed);
const defaultMinLength = hasCJK ? 6 : 15;
```

**2. Multi-Pass Metadata Stripping**
The `stripLeadingInboundMetadata` function runs up to 6 iterations, stopping when stable. This handles cases where multiple metadata blocks are stacked.

**3. Force-Retrieve Override Before Length Check**
Memory-related queries (e.g., "你记得吗" — "do you remember?") are ALWAYS retrieved even if short. This prevents CJK users from having short recall queries skipped.

### Hook Integration Points

Based on the architecture, these modules are integrated via OpenClaw plugin hooks in `index.ts`:

| Hook | Module | Purpose |
|------|--------|---------|
| `agent_end` | SmartExtractor | Trigger extraction after conversation |
| `before_prompt_build` | adaptive-retrieval + auto-capture-cleanup | Decide if retrieval needed and normalize |
| (implicit) | noise-filter | Pre-filter before extraction |

### Technical Debt and Concerns

1. **Pattern-based detection is brittle** — New phrasing of denials or meta-questions won't be caught. A semantic classifier would be more robust but slower.

2. **No cross-session deduplication** — Auto-capture dedup only runs per-extraction batch. If the same memory is captured across multiple sessions before dedup runs, it may be stored multiple times.

3. **Metadata stripping relies on string matching** — If OpenClaw changes envelope format (even slightly), stripping breaks and metadata leaks into extraction.

4. **No retrieval result caching** — Adaptive retrieval decides to skip/retrieve but doesn't cache the retrieval results. If the same query is made twice in quick succession, both hit the embedding API.

5. **Skip patterns may false-positive on short CJK** — The CJK length threshold of 6 could miss meaningful short queries like "叫什么" (what's it called).

6. **Force-retrieve patterns are English-centric** — The FORCE_RETRIEVE_PATTERNS include English bias ("remember", "recall", "my name"). While CJK patterns exist, they may not cover all cultural phrasing variants.
