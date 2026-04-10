# My Action Items: Building a Similar Memory/Vector Database Project

**Derived from:** 12-lessons-learned.md analysis
**Priority:** High (day-one), Medium (before feature development), Low (ongoing improvement)
**Date:** 2026-03-27

---

## Phase 1: Day-One Tooling (Non-Negotiable)

These items must be in place before writing any feature code. They prevent technical debt from accumulating.

### 1.1 Add TypeScript Configuration

**File:** Create `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUncheckedIndexedAccess": true,
    "forceConsistentCasingInFileNames": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**Why:** Prevents the 126 `any` escape hatches observed in memory-lancedb-pro. Forces explicit handling of external SDK types.

**Alternative for external SDKs:** Use `unknown` with explicit type guards or casting, not `any`.

---

### 1.2 Add ESLint + Prettier

**Files:** Create `.eslintrc.js`, `.prettierrc`

**.eslintrc.js:**
```javascript
export default [
  {
    files: ["**/*.ts"],
    languageOptions: {
      ecmaVersion: 2022,
      sourceType: "module",
    },
    plugins: {
      "@typescript-eslint": "...",
    },
    rules: {
      "@typescript-eslint/no-explicit-any": "error",
      "@typescript-eslint/explicit-function-return-type": "off",
      "@typescript-eslint/no-unused-vars": ["error", { argsIgnorePattern: "^_" }],
    },
  },
];
```

**.prettierrc:**
```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "es5",
  "printWidth": 100,
  "tabWidth": 2,
  "arrowParens": "always"
}
```

**Pre-commit hook (package.json):**
```json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged"
    }
  },
  "lint-staged": {
    "*.ts": ["eslint --fix", "prettier --write"]
  }
}
```

**Why:** memory-lancedb-pro has 44 modules with inconsistent formatting. No automated enforcement. Lint-staged ensures every commit is clean.

---

### 1.3 Define Constants Module

**File:** Create `src/constants.ts`

```typescript
/**
 * Retrieval weights for hybrid search fusion.
 * Vector weight 0.7 + BM25 weight 0.3 = 1.0
 */
export const VECTOR_WEIGHT = 0.7;
export const BM25_WEIGHT = 0.3;

/**
 * Score preservation floors based on BM25 relevance.
 * High BM25 hits (>=0.75) should not be heavily demoted by reranking.
 */
export const BM25_HIGH_FLOOR = 0.95;
export const BM25_MEDIUM_FLOOR = 0.9;
export const BM25_LOW_FLOOR = 0.5;

/**
 * Reranking blend weights.
 * 60% cross-encoder score + 40% original fused score.
 */
export const CROSS_ENCODER_BLEND = 0.6;
export const ORIGINAL_SCORE_BLEND = 0.4;

/**
 * Cosine fallback blend (when reranking API fails).
 */
export const COSINE_FALLBACK_BLEND = 0.7;
export const ORIGINAL_FALLBACK_BLEND = 0.3;

/**
 * Embedding cache configuration.
 * 256 entries provides good hit rate without excessive memory.
 * 30-minute TTL balances freshness with API call reduction.
 */
export const EMBEDDING_CACHE_MAX_SIZE = 256;
export const EMBEDDING_CACHE_TTL_MS = 30 * 60 * 1000;

/**
 * Chunking strategy for long documents.
 * - 70% of context limit: conservative to avoid truncation
 * - 5% overlap: preserves cross-chunk context
 * - 10% minimum: ensures chunks aren't too small
 */
export const CHUNK_SIZE_RATIO = 0.7;
export const CHUNK_OVERLAP_RATIO = 0.05;
export const CHUNK_MIN_RATIO = 0.1;
export const CJK_CHUNK_DIVISOR = 2.5;  // CJK chars ~= 2-3 tokens each
export const CJK_RATIO_THRESHOLD = 0.3; // Trigger CJK divisor above this ratio

/**
 * Recency boost parameters.
 * 14-day half-life means memory from 2 weeks ago gets ~50% recency boost.
 */
export const RECENCY_HALF_LIFE_DAYS = 14;
export const RECENCY_WEIGHT = 0.1;

/**
 * Time decay parameters for Weibull distribution.
 * Beta values by tier:
 * - Core (0.8): Stretched exponential - slow decay
 * - Working (1.0): Standard exponential
 * - Peripheral (1.3): Super-exponential - fast decay
 */
export const WEIBULL_BETA_CORE = 0.8;
export const WEIBULL_BETA_WORKING = 1.0;
export const WEIBULL_BETA_PERIPHERAL = 1.3;

/**
 * Admission control weights (A-MAC style).
 */
export const ADMISSION_TYPE_PRIOR_WEIGHT = 0.6;
export const ADMISSION_UTILITY_WEIGHT = 0.1;
export const ADMISSION_CONFIDENCE_WEIGHT = 0.1;
export const ADMISSION_NOVELTY_WEIGHT = 0.1;
export const ADMISSION_RECENCY_WEIGHT = 0.1;
export const NOVELTY_THRESHOLD = 0.55;

/**
 * Composite score decay floors by tier.
 * Core memories nearly never forgotten (0.9 floor).
 */
export const TIER_FLOOR_CORE = 0.9;
export const TIER_FLOOR_WORKING = 0.7;
export const TIER_FLOOR_PERIPHERAL = 0.5;
```

**Why:** memory-lancedb-pro has 126 magic numbers scattered across files. A constants module makes tuning possible and documents rationale.

---

### 1.4 Set Up Test Infrastructure with Coverage

**Package additions:**
```bash
npm install -D vitest @vitest/coverage-v8 c8
```

**File:** Create `vitest.config.ts`

```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    globals: true,
    environment: "node",
    include: ["src/**/*.test.ts", "test/**/*.test.ts"],
    coverage: {
      provider: "v8",
      reporter: ["text", "lcov", "html"],
      include: ["src/**/*.ts"],
      exclude: ["src/**/*.d.ts"],
      thresholds: {
        statements: 70,
        branches: 70,
        functions: 70,
        lines: 70,
      },
    },
  },
});
```

**CI integration:** Add coverage gate to prevent regression below 70%.

**Why:** memory-lancedb-pro has 49 test files but no coverage reporting. You cannot measure quality improvement without measurement.

---

## Phase 2: Architecture Foundations (Before Feature Development)

These architectural decisions should be made before implementing core features.

### 2.1 Choose Layered Architecture with Plugin Integration

**Pattern to follow:** Same as memory-lancedb-pro

```
OpenClaw Plugin Layer (index.ts)
         │
         ▼
   Tool/Service Layer (tools.ts)
         │
         ▼
   Retrieval Pipeline (retriever.ts)
         │
    ┌────┴────┐
    ▼         ▼
 Embedder   Storage
(embedder)  (store)
```

**Key interfaces to define early:**
- `MemoryStore` interface with scope filtering
- `MemoryRetriever` interface with hybrid retrieval
- `Embedder` interface with caching and key rotation
- `ScopeManager` interface with ACL-based access

**Why:** The layered approach in memory-lancedb-pro scales to 12 features without architectural rot. Clear interfaces enable testing and replacement.

---

### 2.2 Implement Serialized Update Queue

**Pattern from memory-lancedb-pro (`store.ts`):**

```typescript
private updateQueue: Promise<void> = Promise.resolve();

private async runSerializedUpdate<T>(action: () => Promise<T>): Promise<T> {
  const previous = this.updateQueue;
  let release: (() => void) | undefined;
  const lock = new Promise<void>((resolve) => { release = resolve; });
  this.updateQueue = previous.then(() => lock);
  await previous;
  try {
    return await action();
  } finally {
    release?.();
  }
}
```

**Why:** Prevents race conditions when multiple async operations update the same memory. Essential for correctness.

---

### 2.3 Implement Cross-Process File Locking

**Pattern from memory-lancedb-pro (`store.ts`):**

```typescript
import properLockfile from "proper-lockfile";

private async runWithFileLock<T>(fn: () => Promise<T>): Promise<T> {
  const release = await lockfile.lock(this.dbPath, {
    retries: { retries: 5, factor: 2, minTimeout: 100, maxTimeout: 2000 },
    stale: 10000,
  });
  try {
    return await fn();
  } finally {
    await release();
  }
}
```

**Why:** Multiple OpenClaw instances may access the same database. File locking prevents corruption.

---

### 2.4 Define Error Class Hierarchy

**File:** Create `src/errors.ts`

```typescript
export class MemoryError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "MemoryError";
  }
}

export class MemoryNotFoundError extends MemoryError {
  constructor(id: string) {
    super(`Memory not found: ${id}`);
    this.name = "MemoryNotFoundError";
  }
}

export class EmbeddingTimeoutError extends MemoryError {
  constructor(provider: string) {
    super(`Embedding request timed out: ${provider}`);
    this.name = "EmbeddingTimeoutError";
  }
}

export class ScopeAccessDeniedError extends MemoryError {
  constructor(scope: string, agentId: string) {
    super(`Access denied to scope '${scope}' for agent '${agentId}'`);
    this.name = "ScopeAccessDeniedError";
  }
}

export class InvalidScopeError extends MemoryError {
  constructor(scope: string) {
    super(`Invalid scope: ${scope}`);
    this.name = "InvalidScopeError";
  }
}
```

**Why:** memory-lancedb-pro uses plain `Error` everywhere. Custom error classes enable precise error handling and categorization.

---

## Phase 3: Core Feature Implementation

### 3.1 Implement Hybrid Retrieval First

**Why:** This is the core differentiator. Should be stable before layering on extraction and lifecycle features.

**Key implementation points:**
1. Vector search with cosine similarity
2. BM25 search via FTS index
3. RRF fusion with configurable weights
4. Ghost entry detection
5. Score preservation floors for high-BM25 hits

**Reference:** `src/retriever.ts` (lines 617-827), `src/store.ts` (lines 479-695)

---

### 3.2 Implement Scope Isolation

**Why:** Security foundation. Should be in place before multi-agent scenarios.

**Key implementation points:**
1. Scope patterns: global, agent:<id>, project:<id>, user:<id>, custom:<name>, reflection:agent:<id>
2. System bypass IDs: "system", "undefined"
3. ACL-based access control with configurable per-agent access lists
4. Write scope resolution (default to agent's own scope)
5. Read filter semantics: undefined=bypass, []=deny-all, [scopes]=filter

**Reference:** `src/scopes.ts` (lines 188-251)

---

### 3.3 Implement LRU Embedding Cache

**Why:** Performance critical. Every retrieval hits embeddings.

**Key implementation points:**
1. SHA-256 hash key including task type
2. LRU eviction when cache full
3. TTL-based expiration (30 minutes default)
4. Cache hit/miss tracking for diagnostics

**Reference:** `src/embedder.ts` (lines 24-80)

---

### 3.4 Implement Admission Control

**Why:** Quality gate before expensive deduplication.

**Key implementation points:**
1. Five-component weighted scoring (typePrior, utility, confidence, novelty, recency)
2. Reject/admit/hold decision logic
3. Audit trail with all feature scores
4. Three presets: balanced, conservative, high-recall

**Reference:** `src/admission-control.ts` (lines 666-747)

---

## Phase 4: Feature Enhancement

### 4.1 Implement Cross-Encoder Reranking

**After:** Hybrid retrieval is stable

**Key implementation points:**
1. Provider adapter pattern (Jina, Voyage, Pinecone, etc.)
2. Request/response normalization per provider
3. 5-second timeout via AbortController
4. Cosine similarity fallback on failure
5. Score blending with preservation floors

**Reference:** `src/retriever.ts` (lines 175-335, 833-959)

---

### 4.2 Implement Memory Lifecycle (Weibull Decay)

**After:** Storage and retrieval are stable

**Key implementation points:**
1. Weibull stretched-exponential decay
2. Tier-specific beta parameters (Core=0.8, Working=1.0, Peripheral=1.3)
3. Composite scoring: 0.4 recency + 0.3 frequency + 0.3 importance
4. Tier promotion/demotion rules
5. Access tracker with debounced write-back

**Reference:** `src/decay-engine.ts`, `src/tier-manager.ts`, `src/access-tracker.ts`

---

### 4.3 Implement Smart Extraction

**After:** Lifecycle is understood

**Key implementation points:**
1. Six categories: profile, preferences, entities, events, cases, patterns
2. L0/L1/L2 layered storage (abstract, overview, content)
3. Two-stage deduplication: cosine pre-filter + LLM decision
4. Envelope metadata stripping (OpenClaw-specific)
5. Admission control integration

**Reference:** `src/smart-extractor.ts`, `src/extraction-prompts.ts`, `src/smart-metadata.ts`

---

## Phase 5: Polish and Operations

### 5.1 Implement Comprehensive CLI

**Commands to implement:**
- `list` - list memories with scope/category filtering
- `search <query>` - hybrid retrieval search
- `stats` - aggregate statistics
- `delete <id>` - single memory deletion
- `delete-bulk` - bulk delete with dry-run
- `export` - JSON export (exclude vectors for size)
- `import <file>` - import with idempotency
- `reembed` - re-embed from source to target
- `upgrade` - upgrade legacy format
- `migrate check/run/verify` - migration workflow
- `auth login/logout` - OAuth authentication
- `reindex-fts` - rebuild FTS index

**Reference:** `cli.ts` (lines 649-1340)

---

### 5.2 Add Structured Logging

**Pattern to implement:**

```typescript
interface Logger {
  debug: (message: string, meta?: Record<string, unknown>) => void;
  info: (message: string, meta?: Record<string, unknown>) => void;
  warn: (message: string, meta?: Record<string, unknown>) => void;
  error: (message: string, meta?: Record<string, unknown>) => void;
}

// Injected like memory-lancedb-pro but with levels
this.logger.info("retrieval:completed", {
  queryLength: query.length,
  resultCount: results.length,
  durationMs: Date.now() - start,
  cacheHit: false,
});
```

**Why:** memory-lancedb-pro mixes console.log and injected logger with no levels. Structured logging enables log aggregation and analysis.

---

### 5.3 Implement Observability

**Tracing pattern to adopt:**

```typescript
trace?.startStage("vector_search", []);
const vectorResults = await this.store.vectorSearch(queryEmbedding, limit);
trace?.endStage(
  vectorResults.map(r => r.id),
  vectorResults.map(r => r.score)
);
```

**Metrics to track:**
- Cache hit rate
- Embedding latency
- Retrieval latency per stage
- Memory count per scope
- Admission rejection rate
- Tier distribution

---

## Ongoing: Technical Debt Management

### File Size Limits

Set maximum file sizes and enforce via lint rule:
- Warning at 500 lines
- Error at 800 lines
- Break large files before they become unmaintainable

### Magic Number Detection

ESLint rule to flag numeric literals without explanation:

```typescript
// Bad
const score = item.score * 0.6 + original.score * 0.4;

// Good
const score = item.score * CROSS_ENCODER_BLEND + original.score * ORIGINAL_SCORE_BLEND;
```

### Regular Dependency Updates

memory-lancedb-pro is at beta.10 with 376 commits in March 2026. Stay current to avoid upgrade debt.

---

## Summary: Prioritized Action Items

| Priority | Item | Why |
|----------|------|-----|
| P0 | Add tsconfig.json with strict mode | Prevents 126 `any` escape hatches |
| P0 | Add ESLint + Prettier + pre-commit hooks | Enforces code quality from day one |
| P0 | Create constants.ts with documented magic numbers | Makes tuning possible |
| P0 | Set up Vitest with coverage reporting | Measures what matters |
| P1 | Implement serialized update queue | Correctness foundation |
| P1 | Implement cross-process file locking | Safety for multi-instance deployments |
| P1 | Define error class hierarchy | Enables precise error handling |
| P1 | Implement hybrid retrieval (vector + BM25) | Core differentiator |
| P1 | Implement scope isolation | Security foundation |
| P2 | Implement LRU embedding cache | Performance critical |
| P2 | Implement admission control | Quality gate |
| P2 | Implement cross-encoder reranking | After hybrid retrieval |
| P3 | Implement Weibull decay lifecycle | After storage/retrieval stable |
| P3 | Implement smart extraction | After lifecycle understood |
| P3 | Implement comprehensive CLI | Operations readiness |
| P3 | Add structured logging | Operational visibility |

**Total estimated items:** 16 prioritized actions across 5 phases

**Key insight:** memory-lancedb-pro built 12 features on a solid foundation. Resist feature pressure until the foundation (tooling, architecture, core retrieval) is stable.
