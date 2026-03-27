# Feature Deep Dive: Batch 3

**Repository:** `/Users/sheldon/Documents/claw/reference/qmd/`
**Date:** 2026-03-27

## Features Analyzed
1. Query Expansion Model - Fine-tuned Qwen3-1.7B for lex/vec/hyde generation
2. Multi-Format Output - JSON, CSV, Markdown, XML output formats
3. Document Retrieval - Fetch by path, docid, or glob patterns

---

## 1. Query Expansion Model

**Files:** `src/llm.ts` (inference), `finetune/train.py` (training), `finetune/reward.py` (scoring)

### Overview

The query expansion model transforms a raw user query into structured multi-format search variations. Given "auth config", it generates:
- `hyde:` - A hypothetical document passage answering the query
- `lex:` - BM25 keyword variations (short, keyword-focused)
- `vec:` - Semantic reformulations (natural language)

### Model Configuration

**Default generation model** (`src/llm.ts:199`):
```typescript
const DEFAULT_GENERATE_MODEL = "hf:tobil/qmd-query-expansion-1.7B-gguf/qmd-query-expansion-1.7B-q4_k_m.gguf";
```

The model is fine-tuned Qwen3-1.7B on ~2,290 training examples, quantized to Q4_K_M for inference efficiency.

### Inference Pipeline

**expandQuery method** (`src/llm.ts:1013-1100`):

```typescript
async expandQuery(query: string, options: { context?: string, includeLexical?: boolean, intent?: string } = {}): Promise<Queryable[]> {
  const llama = await this.ensureLlama();
  await this.ensureGenerateModel();

  // Create grammar to enforce structured output format
  const grammar = await llama.createGrammar({
    grammar: `
      root ::= line+
      line ::= type ": " content "\\n"
      type ::= "lex" | "vec" | "hyde"
      content ::= [^\\n]+
    `
  });

  const intent = options.intent;
  const prompt = intent
    ? `/no_think Expand this search query: ${query}\nQuery intent: ${intent}`
    : `/no_think Expand this search query: ${query}`;

  // Create bounded context (2048 tokens) to prevent large VRAM allocations
  const genContext = await this.generateModel!.createContext({
    contextSize: this.expandContextSize,  // Default 2048
  });
  const sequence = genContext.getSequence();
  const session = new LlamaChatSession({ contextSequence: sequence });

  // Qwen3 recommended settings for non-thinking mode
  const result = await session.prompt(prompt, {
    grammar,
    maxTokens: 600,
    temperature: 0.7,
    topK: 20,
    topP: 0.8,
    repeatPenalty: { lastTokens: 64, presencePenalty: 0.5 },
  });
  // ...
}
```

### Key Implementation Details

**Grammar-constrained generation** ensures the model outputs valid structured format. The grammar enforces `type: content\n` pattern, preventing malformed responses.

**Bounded context size** (2048 tokens) prevents large VRAM allocations for generation:
```typescript
const genContext = await this.generateModel!.createContext({
  contextSize: this.expandContextSize,  // From config or QMD_EXPAND_CONTEXT_SIZE env var
});
```

**Response parsing** (`src/llm.ts:1061-1079`):
```typescript
const lines = result.trim().split("\n");
const queryTerms = query.toLowerCase().replace(/[^a-z0-9\s]/g, " ").split(/\s+/).filter(Boolean);

const queryables: Queryable[] = lines.map(line => {
  const colonIdx = line.indexOf(":");
  if (colonIdx === -1) return null;
  const type = line.slice(0, colonIdx).trim();
  if (type !== 'lex' && type !== 'vec' && type !== 'hyde') return null;
  const text = line.slice(colonIdx + 1).trim();
  // Filter out expansions that don't contain any query terms
  if (!queryTerms.some(term => text.toLowerCase().includes(term))) return null;
  return { type: type as QueryType, text };
}).filter((q): q is Queryable => q !== null);
```

**Fallback handling** on parse failure or empty results:
```typescript
const fallback: Queryable[] = [
  { type: 'hyde', text: `Information about ${query}` },
  { type: 'lex', text: query },
  { type: 'vec', text: query },
];
```

### Training Pipeline

**SFT-only pipeline** (primary, `finetune/train.py`):
- Base model: Qwen3-1.7B
- Training: Supervised fine-tuning on ~2,290 labeled examples
- GRPO moved to experiments/ (not production)

**Data format** (from `finetune/dataset/schema.py`):
```json
{"query": "auth config", "output": [["hyde", "..."], ["lex", "..."], ["vec", "..."]]}
```

**Scoring rubric** (`finetune/SCORING.md`):
- Structure: +10 for each of lex/vec present, -10 if missing
- Diversity bonus: +5 for multiple diverse lex/vec lines
- Format penalty: -5 per invalid line

### Clever Solutions

1. **Grammar-guided decoding** - The grammar constrains output format at inference time, avoiding post-hoc validation failures.

2. **Query term filtering** - Expansions must contain at least one term from the original query, preventing irrelevant outputs.

3. **IncludeLexical flag** - Allows callers to disable `lex:` expansions (for vector-only searches).

4. **Intent injection** - Optional query intent passed through to guide expansion style.

### Technical Debt

1. **Temperature=0.7 warning** - Code comments note "DO NOT use greedy decoding (temp=0) - causes repetition loops", but 0.7 can still produce variability.

2. **Fixed maxTokens=600** - Not configurable, could truncate longer expansions.

3. **Sequential dispose pattern** - Generation context disposed sequentially after parsing; could use Promise.all for parallel cleanup if multiple generations were needed.

---

## 2. Multi-Format Output

**File:** `src/cli/formatter.ts`

### Overview

The formatter system provides 6 output formats for search results and document retrieval: `json`, `csv`, `md`, `xml`, `files`, and `cli`. Each format has specialized handling for the data structures returned by search and document retrieval operations.

### Format Types

**Supported formats** (`src/cli/formatter.ts:36`):
```typescript
export type OutputFormat = "cli" | "csv" | "md" | "xml" | "files" | "json";
```

### Search Result Formatters

**JSON format** (`src/cli/formatter.ts:97-123`):
```typescript
export function searchResultsToJson(
  results: SearchResult[],
  opts: FormatOptions = {}
): string {
  const output = results.map(row => {
    const bodyStr = row.body || "";
    let body = opts.full ? bodyStr : undefined;
    let snippet = !opts.full ? extractSnippet(bodyStr, query, 300, row.chunkPos, undefined, opts.intent).snippet : undefined;

    return {
      docid: `#${row.docid}`,
      score: Math.round(row.score * 100) / 100,
      file: row.displayPath,
      title: row.title,
      ...(row.context && { context: row.context }),
      ...(body && { body }),
      ...(snippet && { snippet }),
    };
  });
  return JSON.stringify(output, null, 2);
}
```

**CSV format** (`src/cli/formatter.ts:128-152`):
```typescript
export function searchResultsToCsv(
  results: SearchResult[],
  opts: FormatOptions = {}
): string {
  const header = "docid,score,file,title,context,line,snippet";
  const rows = results.map(row => {
    const { line, snippet } = extractSnippet(bodyStr, query, 500, row.chunkPos, undefined, opts.intent);
    let content = opts.full ? bodyStr : snippet;
    return [
      `#${row.docid}`,
      row.score.toFixed(4),
      escapeCSV(row.displayPath),
      escapeCSV(row.title),
      escapeCSV(row.context || ""),
      line,
      escapeCSV(content),
    ].join(",");
  });
  return [header, ...rows].join("\n");
}
```

**Markdown format** (`src/cli/formatter.ts:167-187`):
```typescript
export function searchResultsToMarkdown(
  results: SearchResult[],
  opts: FormatOptions = {}
): string {
  return results.map(row => {
    const heading = row.title || row.displayPath;
    const contextLine = row.context ? `**context:** ${row.context}\n` : "";
    return `---\n# ${heading}\n\n**docid:** \`#${row.docid}\`\n${contextLine}\n${content}\n`;
  }).join("\n");
}
```

**XML format** (`src/cli/formatter.ts:192-208`):
```typescript
export function searchResultsToXml(
  results: SearchResult[],
  opts: FormatOptions = {}
): string {
  const items = results.map(row => {
    const titleAttr = row.title ? ` title="${escapeXml(row.title)}"` : "";
    const contextAttr = row.context ? ` context="${escapeXml(row.context)}"` : "";
    return `<file docid="#${row.docid}" name="${escapeXml(row.displayPath)}"${titleAttr}${contextAttr}>\n${escapeXml(content)}\n</file>`;
  });
  return items.join("\n\n");
}
```

### Document Formatters

**MultiGetFile type** (`src/cli/formatter.ts:19-34`):
```typescript
export type MultiGetFile = {
  filepath: string;
  displayPath: string;
  title: string;
  body: string;
  context?: string | null;
  skipped: false;
} | {
  filepath: string;
  displayPath: string;
  title: string;
  body: string;
  context?: string | null;
  skipped: true;
  skipReason: string;
};
```

Documents have distinct handling for `skipped: true` (file too large) vs `skipped: false` (full content).

### Format Options

**FormatOptions** (`src/cli/formatter.ts:38-44`):
```typescript
export type FormatOptions = {
  full?: boolean;       // Show full document content instead of snippet
  query?: string;       // Query for snippet extraction and highlighting
  useColor?: boolean;   // Enable terminal colors (default: false for non-CLI)
  lineNumbers?: boolean;// Add line numbers to output
  intent?: string;      // Domain intent for snippet extraction disambiguation
};
```

### Escape Helpers

**CSV escaping** (`src/cli/formatter.ts:72-79`):
```typescript
export function escapeCSV(value: string | null | number): string {
  if (value === null || value === undefined) return "";
  const str = String(value);
  if (str.includes(",") || str.includes('"') || str.includes("\n")) {
    return `"${str.replace(/"/g, '""')}"`;
  }
  return str;
}
```

**XML escaping** (`src/cli/formatter.ts:81-88`):
```typescript
export function escapeXml(str: string): string {
  return str
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&apos;");
}
```

### Universal Format Functions

**formatSearchResults** (`src/cli/formatter.ts:381-404`):
```typescript
export function formatSearchResults(
  results: SearchResult[],
  format: OutputFormat,
  opts: FormatOptions = {}
): string {
  switch (format) {
    case "json": return searchResultsToJson(results, opts);
    case "csv": return searchResultsToCsv(results, opts);
    case "files": return searchResultsToFiles(results);
    case "md": return searchResultsToMarkdown(results, opts);
    case "xml": return searchResultsToXml(results, opts);
    case "cli": return searchResultsToMarkdown(results, opts);  // Fallback to md
    default: return searchResultsToJson(results, opts);
  }
}
```

**formatDocuments** (`src/cli/formatter.ts:409-430`):
```typescript
export function formatDocuments(
  results: MultiGetFile[],
  format: OutputFormat
): string {
  switch (format) {
    case "json": return documentsToJson(results);
    case "csv": return documentsToCsv(results);
    case "files": return documentsToFiles(results);
    case "md": return documentsToMarkdown(results);
    case "xml": return documentsToXml(results);
    case "cli": return documentsToMarkdown(results);  // Fallback to md
    default: return documentsToJson(results);
  }
}
```

### Clever Solutions

1. **Snippet extraction integration** - JSON/MD/XML formats use `extractSnippet` from store.ts to show contextually relevant snippets based on query terms.

2. **Optional fields via spread** - Uses `...(condition && { field: value })` pattern to conditionally include fields, keeping output clean.

3. **Line number support** - Common option across formats for debugging/accuracy verification.

4. **Docid prefix** - Always rendered as `#abc123` (with hash) for visual clarity in CLI contexts.

### Technical Debt

1. **CLI format is markdown fallback** - `formatSearchResults` returns markdown for CLI, not truly CLI-optimized colored output. The comment says "CLI format should be handled separately with colors."

2. **Inconsistent score precision** - JSON rounds to 2 decimals, CSV uses 4 decimals, files format uses 2 decimals.

3. **No streaming support** - All formats build complete string in memory before returning.

---

## 3. Document Retrieval

**File:** `src/store.ts`

### Overview

Document retrieval supports fetching documents by virtual path, absolute path, short docid (first 6 characters of content hash), or glob patterns. The system provides fuzzy matching suggestions for typos and handles large file skipping.

### Document Result Type

**DocumentResult** (`src/store.ts:1549-1554`):
```typescript
export type DocumentResult = {
  filepath: string;        // Virtual path (qmd://collection/path)
  displayPath: string;     // Collection-relative path
  title?: string;          // From YAML frontmatter or filename
  context?: string;        // Hierarchical context inherited from parent paths
  hash: string;            // Full content hash (SHA-256)
  docid: string;           // Short docid (first 6 chars of hash)
  collectionName: string;
  modifiedAt: string;
  bodyLength: number;
  body?: string;          // Optional, loaded separately with getDocumentBody
};
```

### Docid System

**getDocid** (`src/store.ts:1559-1561`):
```typescript
export function getDocid(hash: string): string {
  return hash.slice(0, 6);
}
```

**isDocid** (`src/store.ts:2190-2194`):
```typescript
export function isDocid(input: string): boolean {
  const normalized = normalizeDocid(input);
  return normalized.length >= 6 && /^[a-f0-9]+$/i.test(normalized);
}
```

**findDocumentByDocid** (`src/store.ts:2203-2217`):
```typescript
export function findDocumentByDocid(db: Database, docid: string): { filepath: string; hash: string } | null {
  const shortHash = normalizeDocid(docid);
  if (shortHash.length < 1) return null;

  const doc = db.prepare(`
    SELECT 'qmd://' || d.collection || '/' || d.path as filepath, d.hash
    FROM documents d
    WHERE d.hash LIKE ? AND d.active = 1
    LIMIT 1
  `).get(`${shortHash}%`) as { filepath: string; hash: string } | null;

  return doc;
}
```

### Single Document Retrieval

**findDocument** (`src/store.ts:3193-3296`) handles multiple lookup strategies:

```typescript
export function findDocument(db: Database, filename: string, options: { includeBody?: boolean } = {}): DocumentResult | DocumentNotFound {
  let filepath = filename;

  // Strip :line suffix if present
  const colonMatch = filepath.match(/:(\d+)$/);
  if (colonMatch) {
    filepath = filepath.slice(0, -colonMatch[0].length);
  }

  // Check if this is a docid lookup
  if (isDocid(filepath)) {
    const docidMatch = findDocumentByDocid(db, filepath);
    if (docidMatch) {
      filepath = docidMatch.filepath;
    } else {
      return { error: "not_found", query: filename, similarFiles: [] };
    }
  }

  // Try to match by virtual path first
  let doc = db.prepare(`
    SELECT ${selectCols}
    FROM documents d
    JOIN content ON content.hash = d.hash
    WHERE 'qmd://' || d.collection || '/' || d.path = ? AND d.active = 1
  `).get(filepath) as DbDocRow | null;

  // Try fuzzy match by virtual path (LIKE %suffix)
  if (!doc) {
    doc = db.prepare(`
      SELECT ${selectCols}
      FROM documents d
      JOIN content ON content.hash = d.hash
      WHERE 'qmd://' || d.collection || '/' || d.path LIKE ? AND d.active = 1
      LIMIT 1
    `).get(`%${filepath}`) as DbDocRow | null;
  }

  // Try to match by absolute path
  if (!doc && !filepath.startsWith('qmd://')) {
    const collections = getStoreCollections(db);
    for (const coll of collections) {
      if (filepath.startsWith(coll.path + '/')) {
        const relativePath = filepath.slice(coll.path.length + 1);
        doc = db.prepare(`
          SELECT ${selectCols}
          FROM documents d
          JOIN content ON content.hash = d.hash
          WHERE d.collection = ? AND d.path = ? AND d.active = 1
        `).get(coll.name, relativePath) as DbDocRow | null;
        if (doc) break;
      }
    }
  }

  // Return similar files if not found
  if (!doc) {
    const similar = findSimilarFiles(db, filepath, 5, 5);
    return { error: "not_found", query: filename, similarFiles: similar };
  }
  // ...
}
```

### Multi-Document Retrieval

**findDocuments** (`src/store.ts:3352-3456`) supports glob patterns and comma-separated lists:

```typescript
export function findDocuments(
  db: Database,
  pattern: string,
  options: { includeBody?: boolean; maxBytes?: number } = {}
): { docs: MultiGetResult[]; errors: string[] } {
  const isCommaSeparated = pattern.includes(',') && !pattern.includes('*') && !pattern.includes('?');
  const errors: string[] = [];
  const maxBytes = options.maxBytes ?? DEFAULT_MULTI_GET_MAX_BYTES;

  let fileRows: DbDocRow[];

  if (isCommaSeparated) {
    // Handle comma-separated list (docids, paths, etc.)
    const names = pattern.split(',').map(s => s.trim()).filter(Boolean);
    for (const name of names) {
      let doc = db.prepare(`...`).get(name) as DbDocRow | null;
      if (!doc) {
        // Try fuzzy match
        doc = db.prepare(`...`).get(`%${name}`) as DbDocRow | null;
      }
      if (!doc) {
        const similar = findSimilarFiles(db, name, 5, 3);
        errors.push(`File not found: ${name} (did you mean: ${similar.join(', ')}?)`);
      } else {
        fileRows.push(doc);
      }
    }
  } else {
    // Handle glob pattern
    const matched = matchFilesByGlob(db, pattern);
    if (matched.length === 0) {
      errors.push(`No files matched pattern: ${pattern}`);
      return { docs: [], errors };
    }
    const virtualPaths = matched.map(m => m.filepath);
    const placeholders = virtualPaths.map(() => '?').join(',');
    fileRows = db.prepare(`
      SELECT ${selectCols}
      FROM documents d
      JOIN content ON content.hash = d.hash
      WHERE 'qmd://' || d.collection || '/' || d.path IN (${placeholders}) AND d.active = 1
    `).all(...virtualPaths) as DbDocRow[];
  }

  // Process results, skip large files
  for (const row of fileRows) {
    if (row.body_length > maxBytes) {
      results.push({
        doc: { filepath: virtualPath, displayPath: row.display_path },
        skipped: true,
        skipReason: `File too large (${Math.round(row.body_length / 1024)}KB > ${Math.round(maxBytes / 1024)}KB)`,
      });
      continue;
    }
    // ...
  }
}
```

### Glob Matching

**matchFilesByGlob** (`src/store.ts:2234-2254`):
```typescript
export function matchFilesByGlob(db: Database, pattern: string): { filepath: string; displayPath: string; bodyLength: number }[] {
  const allFiles = db.prepare(`
    SELECT
      'qmd://' || d.collection || '/' || d.path as virtual_path,
      LENGTH(content.doc) as body_length,
      d.path,
      d.collection
    FROM documents d
    JOIN content ON content.hash = d.hash
    WHERE d.active = 1
  `).all() as { virtual_path: string; body_length: number; path: string; collection: string }[];

  const isMatch = picomatch(pattern);
  return allFiles
    .filter(f => isMatch(f.virtual_path) || isMatch(f.path))
    .map(f => ({
      filepath: f.virtual_path,
      displayPath: f.path,
      bodyLength: f.body_length
    }));
}
```

Uses `picomatch` library for glob pattern matching against both virtual paths and relative paths.

### Fuzzy Matching

**findSimilarFiles** (`src/store.ts:2219-2232`):
```typescript
export function findSimilarFiles(db: Database, query: string, maxDistance: number = 3, limit: number = 5): string[] {
  const allFiles = db.prepare(`
    SELECT d.path FROM documents d WHERE d.active = 1
  `).all() as { path: string }[];

  const queryLower = query.toLowerCase();
  const scored = allFiles
    .map(f => ({ path: f.path, dist: levenshtein(f.path.toLowerCase(), queryLower) }))
    .filter(f => f.dist <= maxDistance)
    .sort((a, b) => a.dist - b.dist)
    .slice(0, limit);
  return scored.map(f => f.path);
}
```

Uses Levenshtein distance for typo tolerance with configurable max distance (default 3) and result limit (default 5).

### Document Body Retrieval

**getDocumentBody** (`src/store.ts:3302-3346`):
```typescript
export function getDocumentBody(db: Database, doc: DocumentResult | { filepath: string }, fromLine?: number, maxLines?: number): string | null {
  let row: { body: string } | null = null;

  // Try virtual path first
  if (filepath.startsWith('qmd://')) {
    row = db.prepare(`
      SELECT content.doc as body
      FROM documents d
      JOIN content ON content.hash = d.hash
      WHERE 'qmd://' || d.collection || '/' || d.path = ? AND d.active = 1
    `).get(filepath) as { body: string } | null;
  }

  // Try absolute path
  if (!row) {
    const collections = getStoreCollections(db);
    for (const coll of collections) {
      if (filepath.startsWith(coll.path + '/')) {
        const relativePath = filepath.slice(coll.path.length + 1);
        row = db.prepare(`...`).get(coll.name, relativePath) as { body: string } | null;
        if (row) break;
      }
    }
  }

  // Optional line range slicing
  if (fromLine !== undefined || maxLines !== undefined) {
    const lines = body.split('\n');
    const start = (fromLine || 1) - 1;
    const end = maxLines !== undefined ? start + maxLines : lines.length;
    body = lines.slice(start, end).join('\n');
  }

  return body;
}
```

### Clever Solutions

1. **Virtual path priority** - Virtual paths (`qmd://collection/path`) checked first for exact match, then fuzzy suffix match, then falls back to absolute path resolution.

2. **Docid with prefix stripping** - Accepts `#abc123`, `abc123`, `"abc123"` - normalizes to short hash.

3. **Collision handling** - Uses `LIKE 'abc123%'` to find first match when multiple documents share the same 6-char prefix.

4. **Large file skipping** - `multiGet` skips files exceeding `maxBytes` (default 10KB) with reason explanation, allowing batch operations to succeed despite size variation.

5. **Context inheritance** - `getContextForFile` retrieves hierarchical context automatically attached to retrieved documents.

### Technical Debt

1. **Sequential comma-separated lookups** - Each file in comma-separated list is looked up with individual DB prepare/get calls, not batched.

2. **All files loaded for glob** - `matchFilesByGlob` loads ALL active file paths into memory before filtering with picomatch, not pushed to SQLite.

3. **No docid collision reporting** - When multiple documents share a short hash, only the first is returned silently.

4. **Levenshtein on full paths** - `findSimilarFiles` computes edit distance on full paths, not just filename components. A typo in a deep path could score poorly.

---

## Summary

| Feature | Key Implementation | Notable Pattern |
|---------|-------------------|-----------------|
| Query Expansion | Grammar-constrained generation with structured output | Bounded context, query term filtering, fallback handling |
| Multi-Format | 6 formats via switch statement, shared extractSnippet | Optional fields via spread, escape helpers per format |
| Document Retrieval | Virtual path -> absolute path -> docid lookup chain | Fuzzy matching, glob via picomatch, Levenshtein similarity |
