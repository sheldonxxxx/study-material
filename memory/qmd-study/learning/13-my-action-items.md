# My Action Items: Building a Similar Project

**Purpose:** Concrete, prioritized next steps for implementing a local-first knowledge base with hybrid search capabilities.

---

## Phase 1: Foundation (Weeks 1-2)

### 1.1 Architecture Decisions

| Decision | Recommendation | Rationale |
|----------|---------------|-----------|
| **Database** | SQLite + sqlite-vec | Proven in QMD, single-file, ACID-compliant |
| **LLM** | node-llama-cpp with GGUF models | Privacy, no API costs, offline capable |
| **AI Integration** | MCP server as primary interface | Standard protocol for AI agents |
| **Language** | TypeScript strict mode | QMD proves this works at scale |
| **Package Manager** | Bun (with npm compatibility) | Faster installs, native TypeScript |

**Action:** Write ADR-001 documenting these decisions with alternatives considered.

### 1.2 Project Structure

Do NOT create a monolith. Instead:

```
src/
├── db/                    # Database operations, schema, migrations
│   ├── index.ts          # Connection management
│   ├── schema.ts         # Table definitions
│   └── queries/          # Repository functions
├── search/               # Search pipeline
│   ├── fts.ts            # BM25/FTS5 operations
│   ├── vector.ts         # sqlite-vec operations
│   ├── fusion.ts         # RRF implementation
│   └── rerank.ts         # LLM reranking
├── chunking/             # Document chunking
│   ├── index.ts          # Main chunking logic
│   ├── breakpoints.ts    # Break point scoring
│   └── code-fences.ts    # Code block protection
├── llm/                  # LLM abstraction
│   ├── interface.ts      # LLM interface
│   ├── llama-cpp.ts      # node-llama-cpp implementation
│   └── session.ts        # Session management
├── mcp/                  # MCP server
│   └── server.ts         # MCP protocol implementation
├── config/               # Configuration management
│   ├── yaml.ts           # YAML parsing
│   └── validation.ts     # Zod schemas
└── cli/                  # CLI interface
    └── index.ts          # Command definitions
```

**Anti-pattern to avoid:** Do not create a single 4000+ line `store.ts`.

### 1.3 Tooling Setup

**Must have on Day 1:**

```bash
# Initialize with Bun
bun init

# Add ESLint + Prettier (QMD gap #1)
bun add -d eslint prettier
bunx eslint init
# Configure: 4 spaces, single quotes, 100 char line limit

# Add TypeScript + strict config
bun add -d typescript @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

**ESLint rules to enable:**
```json
{
  "@typescript-eslint/no-unused-vars": "error",
  "@typescript-eslint/no-explicit-any": "error",
  "@typescript-eslint/no-non-null-assertion": "error"
}
```

**Action:** Create `eslint.config.mjs` and `.prettierrc` before writing any application code.

### 1.4 Type System Design

**Follow QMD's discriminated union pattern for errors:**

```typescript
// Good pattern from QMD (store.ts)
export type DocumentResult = {
  filepath: string;
  displayPath: string;
  // ...
};

export type DocumentNotFound = {
  error: "not_found";
  query: string;
  similarFiles: string[];
};

export type FindDocumentResult = DocumentResult | DocumentNotFound;
```

**Action:** Create `src/types/index.ts` with all domain types before implementation.

---

## Phase 2: Core Features (Weeks 3-5)

### 2.1 Database Layer

**Implement content-addressable storage (QMD pattern):**

```typescript
// schema.sql
CREATE TABLE content (
  hash TEXT PRIMARY KEY,  -- SHA256
  doc TEXT NOT NULL,
  created_at TEXT NOT NULL
);

CREATE TABLE documents (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  collection TEXT NOT NULL,
  path TEXT NOT NULL,
  title TEXT NOT NULL,
  hash TEXT NOT NULL REFERENCES content(hash),
  created_at TEXT NOT NULL,
  modified_at TEXT NOT NULL,
  active INTEGER NOT NULL DEFAULT 1,
  UNIQUE(collection, path)
);

-- FTS5 for BM25 search
CREATE VIRTUAL TABLE documents_fts USING fts5(
  filepath, title, body,
  tokenize='porter unicode61'
);

-- sqlite-vec for vector search
CREATE VIRTUAL TABLE vectors_vec USING vec0(
  hash_seq TEXT PRIMARY KEY,
  embedding float[384] distance_metric=cosine
);
```

**Triggers for auto-sync (QMD pattern):**

```sql
CREATE TRIGGER documents_ai AFTER INSERT ON documents
WHEN new.active = 1
BEGIN
  INSERT INTO documents_fts(rowid, filepath, title, body)
  SELECT new.id, new.collection || '/' || new.path, new.title,
         (SELECT doc FROM content WHERE hash = new.hash)
  WHERE new.active = 1;
END;
```

**Action:** Implement `src/db/index.ts` with:
- WAL journal mode
- Proper PRAGMA settings (cache_size, temp_store)
- Migration runner
- Repository functions (not a monolith)

### 2.2 Smart Chunking

**Implement break point scoring (QMD pattern):**

```typescript
const BREAK_PATTERNS: [RegExp, number, BreakType][] = [
  [/\n#{1}(?!#)/g, 100, 'h1'],
  [/\n#{2}(?!#)/g, 90, 'h2'],
  [/\n```/g, 80, 'codeblock'],
  [/\n\n+/g, 20, 'blank'],
];

// Never split inside code fences
function findCodeFences(text: string): CodeFenceRegion[] {
  // QMD pattern: track in/out state with match indices
}

// Squared distance decay for optimal cut selection
function findBestCutoff(
  breakPoints: BreakPoint[],
  targetCharPos: number,
  windowChars: number = 800,
  decayFactor: number = 0.7
): number {
  // Higher score closer to target wins
}
```

**Configuration:**
- CHUNK_SIZE_TOKENS = 900
- CHUNK_OVERLAP_TOKENS = 135 (15%)
- CHUNK_WINDOW_TOKENS = 200

**Action:** Write `src/chunking/index.ts` and `src/chunking/breakpoints.ts` with unit tests.

### 2.3 Hybrid Search Pipeline

**Implement staged retrieval:**

```
1. BM25 probe → strong signal detection (score >= 0.85, gap >= 0.15)
2. If no signal → LLM query expansion (lex/vec/hyde)
3. Parallel BM25 + vector retrieval
4. RRF fusion (k=60, weighted)
5. Chunk selection per candidate (keyword overlap)
6. LLM reranking
7. Position-aware blending
```

**RRF formula (from QMD):**

```typescript
function reciprocalRankFusion(
  resultLists: RankedResult[][],
  weights: number[] = [],
  k: number = 60
): RankedResult[] {
  const scores = new Map<string, { result: RankedResult; rrfScore: number }>();

  for (let listIdx = 0; listIdx < resultLists.length; listIdx++) {
    const weight = weights[listIdx] ?? 1.0;
    for (let rank = 0; rank < list.length; rank++) {
      const rrfContribution = weight / (k + rank + 1);
      // Accumulate scores across lists
    }
  }
}
```

**Position-aware blending (QMD pattern):**

```typescript
// Top 3: 75% RRF, 25% rerank
// Ranks 4-10: 60% RRF, 40% rerank
// Ranks 11+: 40% RRF, 60% rerank
```

**Action:** Implement `src/search/fusion.ts` and `src/search/rerank.ts` with evaluation tests.

### 2.4 LLM Session Management

**Reference counting pattern (QMD):**

```typescript
class LLMSessionManager {
  private _activeSessionCount = 0;
  private _inFlightOperations = 0;

  acquire(): void { _activeSessionCount++; }
  release(): void { _activeSessionCount = Math.max(0, _activeSessionCount - 1); }

  canUnload(): boolean {
    return _activeSessionCount === 0 && _inFlightOperations === 0;
  }
}
```

**Inactivity timeout:**
```typescript
const DEFAULT_INACTIVITY_TIMEOUT_MS = 5 * 60 * 1000;
```

**Action:** Implement `src/llm/session.ts` following QMD's pattern.

---

## Phase 3: API and Integration (Weeks 6-7)

### 3.1 MCP Server

**Tool definitions:**

```typescript
// Query tool (primary)
server.registerTool("query", {
  title: "Query",
  inputSchema: {
    searches: z.array(subSearchSchema).min(1).max(10),
    limit: z.number().optional().default(10),
    minScore: z.number().optional().default(0),
  },
}, async ({ searches, limit, minScore }) => {
  const results = await search.hybridQuery(searches, { limit, minScore });
  return {
    content: [{ type: "text", text: formatResults(results) }],
    structuredContent: { results },
  };
});
```

**Transport options:**
- Stdio (default, for CLI integration)
- HTTP with session-per-client (for remote access)

**Action:** Implement `src/mcp/server.ts` with Zod validation for inputs.

### 3.2 SDK Design

**Facade pattern (QMD):**

```typescript
// src/index.ts
export interface QMDStore {
  search(options: SearchOptions): Promise<SearchResult[]>;
  get(pathOrDocid: string, options?: GetOptions): Promise<DocumentResult>;
  multiGet(pattern: string, options?: MultiGetOptions): Promise<MultiGetResult[]>;
  addCollection(name: string, opts: CollectionOptions): Promise<void>;
  update(options?: UpdateOptions): Promise<UpdateResult>;
  embed(options?: EmbedOptions): Promise<EmbedResult>;
  close(): Promise<void>;
}

export async function createStore(options: StoreOptions): Promise<QMDStore> {
  // Create internal store
  // Set up LlamaCpp instance
  // Return QMDStore facade
}
```

**Action:** Implement `src/index.ts` following the facade pattern.

---

## Phase 4: Configuration and Polish (Week 8)

### 4.1 Configuration with Zod

**All configuration must have Zod schemas:**

```typescript
// src/config/validation.ts
import { z } from "zod";

export const CollectionSchema = z.object({
  path: z.string().min(1),
  pattern: z.string().default("**/*.md"),
  ignore: z.array(z.string()).optional(),
  context: z.record(z.string()).optional(),
  includeByDefault: z.boolean().default(true),
});

export const ConfigSchema = z.object({
  global_context: z.string().optional(),
  collections: z.record(CollectionSchema),
});

export function validateConfig(raw: unknown): Config {
  return ConfigSchema.parse(raw);
}
```

**Action:** Add Zod validation to all configuration inputs (collections, CLI args, env vars).

### 4.2 Structured Logging

**Replace console.* with structured logger:**

```typescript
// src/lib/logger.ts
enum LogLevel { DEBUG = 0, INFO = 1, WARN = 2, ERROR = 3 }

interface LogEntry {
  timestamp: string;
  level: LogLevel;
  message: string;
  correlationId?: string;
  metadata?: Record<string, unknown>;
}

class Logger {
  private minLevel: LogLevel;
  private entries: LogEntry[] = [];

  constructor(minLevel: LogLevel = LogLevel.INFO) {
    this.minLevel = minLevel;
  }

  info(message: string, metadata?: Record<string, unknown>): void {
    this.log(LogLevel.INFO, message, metadata);
  }

  error(message: string, error?: Error, metadata?: Record<string, unknown>): void {
    this.log(LogLevel.ERROR, message, { error: error?.message, stack: error?.stack, ...metadata });
  }

  private log(level: LogLevel, message: string, metadata?: Record<string, unknown>): void {
    if (level < this.minLevel) return;
    this.entries.push({
      timestamp: new Date().toISOString(),
      level,
      message,
      metadata,
    });
    // Also emit to stderr for errors
    if (level >= LogLevel.ERROR) {
      console.error(JSON.stringify(this.entries.at(-1)));
    }
  }
}
```

**Action:** Replace all `console.*` calls with structured logging.

### 4.3 Error Handling

**No silent catch blocks:**

```typescript
// BAD (QMD anti-pattern)
try { mkdirSync(dir, { recursive: true }); } catch { }

// GOOD
try {
  mkdirSync(dir, { recursive: true });
} catch (err) {
  throw new Error(`Failed to create directory ${dir}`, { cause: err });
}
```

**Action:** Audit codebase for silent catch blocks and fix all of them.

---

## Phase 5: Testing and Documentation (Week 9)

### 5.1 Test Coverage

**Target: 16+ test files like QMD**

| Test Type | Files | Purpose |
|-----------|-------|---------|
| Unit | `chunking.test.ts` | Break point scoring, chunk boundaries |
| Unit | `fusion.test.ts` | RRF formula |
| Unit | `session.test.ts` | Reference counting |
| Integration | `db.test.ts` | Repository operations |
| Integration | `search.test.ts` | Full search pipeline |
| Integration | `mcp.test.ts` | MCP server |
| Integration | `cli.test.ts` | CLI commands |

**Test isolation pattern:**

```typescript
import { beforeEach, afterEach } from "vitest";

describe("search", () => {
  const testDbPath = `./test-${Date.now()}.sqlite`;

  beforeEach(() => {
    // Set up isolated test database
    process.env.INDEX_PATH = testDbPath;
  });

  afterEach(() => {
    // Clean up
    delete process.env.INDEX_PATH;
    try { unlinkSync(testDbPath); } catch {}
  });
});
```

**Action:** Create comprehensive test suite with >80% coverage on core modules.

### 5.2 Documentation

**Required docs:**

| Document | Purpose |
|----------|---------|
| `README.md` | Quick start, architecture diagram |
| `ARCHITECTURE.md` | Module responsibilities, data flow |
| `API.md` | SDK reference |
| `CONTRIBUTING.md` | Development setup, PR process |

**Action:** Write all documentation before public release.

---

## Critical Pitfalls to Avoid

### 1. Monolithic Store File

**Problem:** QMD's `store.ts` at 4,200 lines makes maintenance difficult.

**Prevention:** Create modules from day one. If a file exceeds 500 lines, split it.

### 2. Skipping ESLint/Prettier

**Problem:** Code quality degrades without enforcement.

**Prevention:** Set up linting and formatting on day 1. Fail CI if linting fails.

### 3. Silent Error Handling

**Problem:** Silent catch blocks hide failures.

**Prevention:** No `catch {}` patterns. All errors must be logged or thrown with context.

### 4. No Runtime Validation

**Problem:** TypeScript types only exist at compile time.

**Prevention:** Use Zod schemas for all external input (config, CLI args, API requests).

### 5. Console Logging

**Problem:** No structured logging makes debugging difficult.

**Prevention:** Implement structured logging from day 1.

---

## Tooling Checklist

| Tool | Purpose | When |
|------|---------|------|
| Bun | Package manager | Day 1 |
| TypeScript (strict) | Language | Day 1 |
| ESLint + Prettier | Code quality | Day 1 |
| Vitest | Testing | Day 1 |
| Zod | Runtime validation | Day 1 |
| sqlite-vec | Vector search | Week 3 |
| node-llama-cpp | LLM inference | Week 4 |
| @modelcontextprotocol/sdk | MCP server | Week 6 |

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Lines of code per module | < 500 |
| Test coverage | > 80% |
| ESLint errors | 0 |
| Console.* calls | 0 (structured logging) |
| Silent catch blocks | 0 |
| any[] casts | 0 |
| Non-null assertions | 0 |

---

## Key Architectural Decisions to Document

1. **SQLite + sqlite-vec** - Why not PostgreSQL with pgvector?
2. **Local GGUF models** - Why not OpenAI/Anthropic API?
3. **MCP as primary interface** - Why not REST/WebSocket?
4. **Content-addressable storage** - Why hash-based deduplication?
5. **Chunk-level reranking** - Why not full document reranking?
6. **Session-based LLM management** - Why reference counting?

Document each decision with alternatives considered and trade-offs made.
