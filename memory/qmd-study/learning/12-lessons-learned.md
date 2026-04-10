# Lessons Learned: Building a Local-First Knowledge Base Search System

**Project:** QMD (Query Markup Documents) Analysis
**Study Date:** 2026-03-27
**Purpose:** Synthesize lessons for building a similar local-first knowledge base with hybrid search

---

## What to EMULATE

### 1. Hybrid Search Pipeline Architecture

The combination of BM25 + vector similarity + Reciprocal Rank Fusion (RRF) + LLM reranking produces significantly better results than any single approach.

**Key insight from code (store.ts:3733-3746):** The "strong signal bypass" skips expensive LLM expansion when BM25 clearly matches (score >= 0.85 with gap >= 0.15). This optimization prevents unnecessary latency for obvious queries.

```
Query Input
    │
    ▼
BM25 Probe (Strong Signal Check)
    │ No signal
    ▼
Query Expansion (LLM generates lex/vec/hyde variants)
    │
    ├──► FTS5 (BM25) for 'lex' queries
    └──► Batch Embed + sqlite-vec for 'vec'/'hyde' queries
              │
              ▼
RRF Fusion (k=60, first 2 lists get 2x weight)
    │
    ▼
Chunk Selection (keyword overlap scoring)
    │
    ▼
LLM Reranking (Qwen3-reranker)
    │
    ▼
Position-Aware Blending (Top 3: 75% RRF / 25% rerank)
```

**Why it works:** Each retrieval method has different failure modes. BM25 excels at exact matches, vectors capture semantic similarity, and LLM reranking refines relevance using full context.

### 2. Smart Document Chunking with Break Point Scoring

QMD scores potential chunk boundaries rather than using fixed token limits (store.ts:97-110):

```typescript
export const BREAK_PATTERNS: [RegExp, number, string][] = [
  [/\n#{1}(?!#)/g, 100, 'h1'],     // H1 headings score 100
  [/\n#{2}(?!#)/g, 90, 'h2'],      // H2 headings score 90
  [/\n```/g, 80, 'codeblock'],      // Code blocks score 80
  [/\n\n+/g, 20, 'blank'],         // Paragraph breaks score 20
  [/\n/g, 1, 'newline'],             // Newlines score 1
];
```

**Critical protection:** `findCodeFences()` ensures code blocks are never split (store.ts:144-166). This preserves code integrity even when chunk boundaries would otherwise fall inside a fence.

**Distance decay formula (store.ts:442-444):** When finding the best cut point, squared distance decay prevents arbitrary cuts near chunk edges:
```typescript
const normalizedDist = distance / windowChars;
const multiplier = 1.0 - (normalizedDist * normalizedDist) * decayFactor;
```

### 3. Content-Addressable Storage

Documents are deduplicated by SHA256 hash (store.ts:659-665):

```typescript
db.exec(`
  CREATE TABLE IF NOT EXISTS content (
    hash TEXT PRIMARY KEY,  -- SHA256 of content
    doc TEXT NOT NULL,
    created_at TEXT NOT NULL
  )
`);
```

**Benefits observed:**
- Same content indexed once regardless of collection membership
- Hash-based docids (`abc123` = first 6 chars of hash) provide collision-resistant short identifiers
- Orphan cleanup via `DELETE FROM content WHERE hash NOT IN (SELECT DISTINCT hash FROM documents)`

### 4. Session-Based LLM Resource Management

The `LLMSessionManager` uses reference counting to track active operations (llm.ts:1276-1502):

```typescript
class LLMSessionManager {
  private _activeSessionCount = 0;
  private _inFlightOperations = 0;

  canUnload(): boolean {
    return this._activeSessionCount === 0 && this._inFlightOperations === 0;
  }
}
```

**Inactivity timeout pattern (llm.ts:162):** Models stay loaded for 5 minutes after last activity, then auto-unload. This balances responsiveness against memory pressure.

### 5. MCP as First-Class Integration

QMD treats the Model Context Protocol server as a primary interface, not an afterthought. The server exposes tools (`query`, `get`, `multi_get`, `status`) and resources (`qmd://` URI scheme) (mcp/server.ts).

**HTTP transport session model (mcp/server.ts:549-572):** Each HTTP client gets its own MCP server instance, but the SQLite store is shared. This follows MCP spec while maintaining efficiency.

**REST endpoint bypass (mcp/server.ts:626-675):** `POST /query` allows structured search without MCP protocol overhead for high-frequency integration scenarios.

### 6. Dual Configuration (YAML + SQLite)

Collections and context are stored in both YAML (human-editable, version-controllable) and SQLite (runtime-efficient) with write-through synchronization (store.ts:921-970).

```typescript
// Config hash prevents unnecessary sync
const hash = createHash('sha256').update(configJson).digest('hex');
if (existingHash?.value === hash) {
  return; // Config unchanged, skip sync
}
```

### 7. Comprehensive Test Suite

16 test files with proper isolation patterns (test/*.test.ts):

```typescript
// Test isolation via INDEX_PATH env var
test("getDefaultDbPath throws in test mode without INDEX_PATH", () => {
  const originalIndexPath = process.env.INDEX_PATH;
  delete process.env.INDEX_PATH;
  expect(() => getDefaultDbPath()).toThrow("Database path not set");
  if (originalIndexPath) {
    process.env.INDEX_PATH = originalIndexPath;
  }
});
```

**CI matrix:** Tests run on Node 22/23 and Bun across Ubuntu and macOS (github/workflows/ci.yml).

### 8. Formal Release Process

Tool-assisted release with changelog validation (skills/release/SKILL.md):
- Pre-push hook validates `package.json` version matches tag
- CHANGELOG.md must have corresponding entry
- `scripts/extract-changelog.sh` generates cumulative release notes

---

## What to AVOID

### 1. No ESLint/Prettier Configuration

**Impact:** The codebase has no consistent style enforcement, no complexity limits, no banned patterns. Score: 3/10 for linting/formatting.

**Specific gaps observed:**
- No import sorting rules
- No max line length enforcement
- No unused variable detection at lint level

### 2. Silent Catch Blocks

**Anti-pattern from store.ts and db.ts:**
```typescript
try { mkdirSync(qmdCacheDir, { recursive: true }); } catch { }
try { BunDatabase.setCustomSQLite(p); break; } catch {}
```

**Problem:** These swallow all errors including permission issues. The mkdir example will silently fail if the directory cannot be created.

### 3. No Structured Logging

**Current state (console-only):**
```typescript
console.error("Embedding error:", error);
console.error("Batch embedding error:", error);
```

**Problems:**
- No log levels (DEBUG, INFO, WARN, ERROR)
- No structured metadata
- No log aggregation support
- No correlation IDs for request tracing

### 4. Monolithic Core File

**store.ts: ~4,200 lines** of core logic including database operations, search, chunking, document management, and context handling.

**Why this is problematic:**
- Makes parallel development difficult (merge conflicts)
- Hard to understand all dependencies
- Single point of failure for bugs
- Delays新人 onboarding

**Better approach:** Separate into `db/`, `search/`, `chunking/`, `context/` modules.

### 5. Disabled TypeScript Strict Flags

**tsconfig.json has three important flags disabled:**
```json
"noUnusedLocals": false,           // Allows unused variables
"noUnusedParameters": false,       // Allows unused params
"noPropertyAccessFromIndexSignature": false  // Allows obj.key on index types
```

### 6. Type Safety Bypasses

**any[] casts observed:**
```typescript
const parsed = JSON.parse(cached) as any[];
```

**Non-null assertions:**
```typescript
const pos = match.index!;  // Uses ! operator to assert existence
```

### 7. Minimal Runtime Validation

Zod is in dependencies but only used in `src/mcp/server.ts`. No Zod schemas for:
- Collection configuration
- CLI arguments
- Environment variables

**Configuration relies on TypeScript types without runtime checks:**
```typescript
const config = YAML.parse(content) as CollectionConfig;  // No validation
```

### 8. Sequential Vector Embedding Limitation

**Comment at store.ts:4040:**
```typescript
// "concurrent embed() hangs" — node-llama-cpp limitation
```

This was a known constraint that required working around rather than solving.

---

## Surprises

### 1. Local LLM Inference Works Well

Running GGUF models via node-llama-cpp provides:
- Privacy (no API calls)
- No API costs
- Offline capability
- Acceptable latency for indexing and reranking

The key insight is treating LLM operations as batch-oriented (indexing) rather than interactive.

### 2. MCP Protocol Enables Clean AI Integration

The Model Context Protocol provides a well-designed interface for AI agent interaction. QMD's implementation shows MCP can be both ergonomic for AI tools and practical for local execution.

**Key insight:** The `qmd://` URI scheme for document resources is elegant - AI agents can reference documents consistently.

### 3. Hash-Based Docids Are Collision-Resistant Enough

Using 6 characters of SHA256 (`abc123`) provides 16^6 = ~16.7 million possibilities with negligible collision risk for personal knowledge bases.

### 4. Single Maintainer Can Move Fast

Despite being a side project, QMD has:
- ~267 commits
- ~44 merged PRs
- ~40 contributors
- v2.0.1 release with semver discipline

The formal release process and tool assistance (changelog validation, pre-push hooks) enable maintainer velocity.

### 5. Chunk-Level Reranking Is Critical Optimization

**From search pipeline comments:** "Reranking full bodies is O(tokens) — unaffordable at scale"

Reranking at the chunk level (900 tokens) rather than full documents (potentially 10K+ tokens) makes LLM reranking practical.

### 6. sqlite-vec Has JOIN Limitations

**From store.ts:2827-2830:**
> "IMPORTANT: We use a two-step query approach here because sqlite-vec virtual tables hang indefinitely when combined with JOINs in the same query."

This required a workaround that separates vector matching from document resolution.

---

## Honest Assessment

### Strengths

| Area | Assessment |
|------|------------|
| TypeScript strictness | Strong - core flags enabled, explicit return types |
| Test coverage | Comprehensive - 16 test files, good isolation |
| Architecture | Sound - layered monolith with clean abstractions |
| Security | Good - parameterized SQL, output encoding, local-only design |
| Performance | Thoughtful - batching, VRAM-aware parallelism, caching |
| Documentation | Excellent - CLAUDE.md, CHANGELOG.md, docs/SYNTAX.md |

### Weaknesses

| Area | Score | Issue |
|------|-------|-------|
| Linting/Formatting | 3/10 | No ESLint/Prettier |
| Error Handling | 6/10 | Silent catch blocks, console.error |
| Config Validation | 5/10 | Minimal runtime validation |
| Logging | 4/10 | No structured logging |
| Type Safety | 7/10 | any[] casts, ! assertions |

**Overall: 5.9/10** — Good foundation with significant tooling gaps.

---

## Key Quotes from Code

### On Chunking
> "Increased from 800 to accommodate smart chunking" (store.ts:51 comment)

### On Reranking Performance
> "Reranking full bodies is O(tokens) — unaffordable at scale" (store.ts comment before chunk selection)

### On sqlite-vec Limitation
> "IMPORTANT: We use a two-step query approach here because sqlite-vec virtual tables hang indefinitely when combined with JOINs in the same query." (store.ts:2827-2830)

### On Temperature Warning
> "DO NOT use greedy decoding (temp=0) - causes repetition loops" (llm.ts:143 comment)
