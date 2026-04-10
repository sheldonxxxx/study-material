# ZeroClaw Feature Deep Dive: Batch 2

**Features covered:** Memory & Knowledge Management, Security & Sandboxing, AI Provider Abstraction
**Source:** `src/memory/sqlite.rs`, `src/security/policy.rs`, `src/providers/mod.rs`, `src/providers/reliable.rs`
**Date:** 2026-03-26

---

## 4. Memory & Knowledge Management

**Key file:** `src/memory/sqlite.rs` (2762 lines)

### Architecture

SQLite-backed brain with hybrid search: vector embeddings + full-text search (FTS5) + BM25 scoring. Single-file persistence at `workspace_dir/memory/brain.db`.

### Hybrid Search Pipeline

The `recall()` method (lines 604-839) runs three search strategies:

1. **BM25 keyword search** via FTS5 virtual table
2. **Vector similarity** via cosine similarity on stored BLOB embeddings
3. **Hybrid merge** with configurable weighting (default 70% vector, 30% keyword)

```rust
// fts5_search returns (id, score) sorted by BM25
let keyword_results = Self::fts5_search(&conn, &query, limit * 2)?;

// vector_search scans all embeddings, computes cosine similarity
let vector_results = Self::vector_search(&conn, qe, limit * 2, None, None)?;

// hybrid_merge combines ranked lists
let merged = vector::hybrid_merge(&vector_results, &keyword_results, vector_weight, keyword_weight, limit)?;
```

**Fallback chain:** If hybrid returns nothing (no embeddings), LIKE-based substring search kicks in.

### Embedding Cache

`get_or_compute_embedding()` (lines 279-339) uses SHA-256 truncated to 16 hex chars as deterministic cache key (avoiding `DefaultHasher` instability across Rust versions):

```rust
fn content_hash(text: &str) -> String {
    use sha2::{Digest, Sha256};
    let hash = Sha256::digest(text.as_bytes());
    format!("{:016x}", u64::from_be_bytes(hash[..8].try_into().expect(...)))
}
```

LRU eviction runs in the same `spawn_blocking` call after insert: deletes oldest-accessed rows until under `cache_max`.

### Schema Design

```sql
CREATE TABLE memories (
    id TEXT PRIMARY KEY, key TEXT NOT NULL UNIQUE,
    content TEXT NOT NULL, category TEXT DEFAULT 'core',
    embedding BLOB, created_at TEXT, updated_at TEXT,
    session_id TEXT, namespace TEXT DEFAULT 'default',
    importance REAL DEFAULT 0.5, superseded_by TEXT
);

CREATE VIRTUAL TABLE memories_fts USING fts5(
    key, content, content=memories, content_rowid=rowid
);
-- Triggers keep FTS in sync on insert/update/delete
```

**Schema migrations** are idempotent checks using `schema_sql.contains()` before ALTERing (lines 206-234). No explicit migration IDs.

### Concurrency & Performance

- WAL mode, `synchronous = NORMAL`, `mmap_size = 8MB`, `cache_size = -2000` (2MB), `temp_store = MEMORY`
- `open_connection()` spawns a thread with timeout (capped at 300s) to avoid blocking on locked DB
- All DB operations use `tokio::task::spawn_blocking` to avoid blocking the async executor
- `Arc<Mutex<Connection>>` sharing: single connection, serialized access

### Session Isolation

Memory entries carry `session_id` and `namespace`. `recall()` filters by session when provided. Sessions cannot cross-contaminate: `SELECT ... WHERE session_id = ?1` in both `recall` and `list`.

### Notable Patterns

- **Upsert via `ON CONFLICT(key)`**: `INSERT INTO memories ... ON CONFLICT(key) DO UPDATE SET content = excluded.content, ...`
- **Single-query multi-ID fetch** (lines 692-705): Instead of N round-trips, builds `WHERE id IN (?1, ?2, ...)` with a single prepared statement
- **Empty/whitespace query** falls back to `recall_by_time_only()` — returns recent entries by `ORDER BY updated_at DESC`
- **`SearchMode` enum**: `Hybrid`, `Bm25`, `Embedding` gates allow pure keyword, pure vector, or combined

### Clever Details

- `store_with_metadata()` supports importance scoring and custom namespaces (lines 1095-1138)
- `superseded_by` column enables soft delete / memory replacement
- `forbidden_path_argument()` detection in security policy uses token-level parsing (attaching short options like `-f/etc/passwd`)

---

## 5. Security & Sandboxing

**Key file:** `src/security/policy.rs` (3128 lines)

### Defense-in-Depth Layers

The policy engine enforces security through five sequential gates:

1. **Autonomy gate** — `ReadOnly` blocks all commands immediately
2. **Subshell/expansion blocker** — blocks `` ` ``, `$(`, `${`, `<(`, `>(`
3. **Redirection blocker** — blocks unquoted `<` and `>`
4. **Tee blocker** — prevents `echo x | tee /file`
5. **Background chain blocker** — blocks bare `&` (but allows `&&`)
6. **Per-segment allowlist** — each sub-command validated independently

### Quote-Aware Shell Lexer

`split_unquoted_segments()` (lines 325-412) parses shell commands respecting quote state:

```rust
enum QuoteState { None, Single, Double }
// `sqlite3 db "SELECT 1; SELECT 2;"` → single segment
// `ls | grep foo` → two segments
// `echo '$(rm -rf /)'` → single segment (literal)
```

This prevents attacks like `echo '$(rm -rf /)'` passing as an allowed `echo` command.

### Command Risk Classification

Per-segment classification (lines 711-832): any high-risk segment marks the whole command high. High-risk commands include `rm`, `curl`, `wget`, `shutdown`, `sudo`, `reboot`, PowerShell, `sc`, `netsh`; and patterns like `rm -rf /`, fork bombs (`:(){:|:&};:`).

Medium-risk: `git commit/push/reset/clean/rebase/merge`, `npm install/add/remove/publish`, `cargo add/remove/install/clean`, `touch/mkdir/mv/cp`.

### Path Validation Layers

`is_path_allowed()` (lines 1123-1185) applies in order:
1. Null byte check
2. `..` component traversal check
3. URL-encoded traversal (`..%2f`)
4. `~user` rejection (but `~/` allowed)
5. Workspace/allowed-root check (if `workspace_only`)
6. Forbidden prefix match

`is_resolved_path_allowed()` (lines 1189-1226) canonicalizes paths and checks after symlink resolution to prevent symlink escapes.

### Explicit Allowlist Exemption

The wildcard `"*"` in `allowed_commands` does NOT exempt from `block_high_risk_commands` (line 862). Explicit entries like `["curl"]` do. This prevents operators who blindly set `*` from bypassing safety.

```rust
let has_wildcard = self.allowed_commands.iter().any(|c| c.trim() == "*");
if has_wildcard && !self.block_high_risk_commands {
    return Ok(risk); // Skip all risk gates
}
// But wildcard alone does NOT bypass:
let explicitly_listed = self.allowed_commands.iter().any(|allowed| {
    // Skip wildcard — does not count as explicit
    if allowed.trim() == "*" { return false; }
    is_allowlist_entry_match(allowed, executable, base_cmd)
});
```

### Rate Limiting

`ActionTracker` (lines 36-78) uses a sliding window with `Instant::now()`:

```rust
pub fn record(&self) -> usize {
    let mut actions = self.actions.lock();
    let cutoff = Instant::now().checked_sub(Duration::from_secs(3600)).unwrap();
    actions.retain(|t| *t > cutoff);
    actions.push(Instant::now());
    actions.len()
}
```

Cloned trackers are independent (each clone gets its own `Mutex<Vec<Instant>>`).

### Runtime Config Protection

`is_runtime_config_path()` (lines 1237-1257) protects `config.toml`, `active_workspace.toml`, and their `.bak` / `.tmp-*` variants in the zeroclaw config directory from agent modification.

### Notable Patterns

- **Windows/Unix parity**: `default_allowed_commands()` and `default_forbidden_paths()` use `#[cfg(target_os = "windows")]` conditionals
- **`rootless_path()`** (lines 260-278): Strips absolute path prefixes to get workspace-relative paths, returning `None` on any `..` component
- **`expand_user_path()`** (lines 244-258): Handles `~` and `~/` expansion across platforms
- **`prompt_summary()`** (lines 1406-1485): Renders human-readable security constraints for LLM system prompt injection (issue #2404)

---

## 6. AI Provider Abstraction & Routing

**Key files:** `src/providers/mod.rs` (provider registry), `src/providers/reliable.rs` (111KB resilient wrapper)

### Provider Registry

`providers/mod.rs` implements a factory pattern with canonical string keys. Aliases resolve to regional or specific endpoints:

```rust
// Regional/provider aliases
is_minimax_intl_alias("minimax") | is_minimax_cn_alias("minimax-cn")
is_glm_alias("glm") | is_glm_cn_alias("glm-cn")
is_moonshot_alias("moonshot") | is_kimi_alias("kimi")
is_qwen_alias("qwen") | is_qwen_intl_alias("qwen-intl")
is_zai_alias("zai") | is_zai_cn_alias("zai-cn")
```

### ReliableProvider: Three-Level Failover

`reliable.rs` wraps any provider with retry/fallback logic. The three nested loops (lines 429-567):

```rust
// Outer: model fallback chain
for current_model in &models {
    // Middle: provider priority list
    for (provider_name, provider) in &self.providers {
        // Inner: retry with exponential backoff
        for attempt in 0..=self.max_retries {
            match provider.chat_with_history(&messages, current_model, temp).await {
                Ok(resp) => return Ok(resp),
                Err(e) => {
                    if is_non_retryable(&e) { break; } // Skip to next provider
                    tokio::time::sleep(Duration::from_millis(wait)).await;
                    backoff_ms = (backoff_ms * 2).min(10_000);
                }
            }
        }
    }
}
```

### Error Classification

The `is_non_retryable()` function (lines 76-139) classifies errors:

- **Retryable**: 5xx, 408, 429, connection errors, model overload
- **Non-retryable**: 4xx except 429/408, auth failures (keyword detection), unknown models

**Context window errors are NOT non-retryable** — they trigger history truncation (drop oldest half of non-system messages) and retry. This recovers without changing models.

**Tool schema errors** (e.g. Groq returning `"tool call validation failed: attempted to call tool 'x' which was not in request"`) are also NOT non-retryable — the provider's built-in fallback in `compatible.rs` can recover by switching to prompt-guided tool instructions.

### Business Logic Rate Limits

`is_non_retryable_rate_limit()` (lines 193-231) detects 429 errors that retries cannot fix:
- `"plan does not include glm-5"`
- `"insufficient balance"`
- Known error codes: `1113`, `1311`

Generic rate limits still get retries with exponential backoff.

### Retry-After Parsing

`parse_retry_after_ms()` (lines 235-264) extracts `Retry-After` from error strings:

```rust
// Matches: "retry-after: 5", "retry_after: 2.5", "Retry-After 7"
fn parse_retry_after_ms(err: &anyhow::Error) -> Option<u64> {
    for prefix in &["retry-after:", "retry_after:", "retry-after ", "retry_after "] {
        if let Some(pos) = lower.find(prefix) {
            let num_str: String = after
                .trim()
                .chars()
                .take_while(|c| c.is_ascii_digit() || *c == '.')
                .collect();
            if let Ok(secs) = num_str.parse::<f64>() {
                return Some(Duration::from_secs_f64(secs).as_millis() as u64);
            }
        }
    }
    None
}
```

### Task-Local Fallback Info

When a fallback occurs (different provider/model than requested), `record_provider_fallback()` (lines 52-67) stores info in a `tokio::task_local!` static:

```rust
tokio::task_local! {
    static PROVIDER_FALLBACK: RefCell<Option<ProviderFallbackInfo>>;
}

pub async fn scope_provider_fallback<F: std::future::Future>(future: F) -> F::Output {
    PROVIDER_FALLBACK.scope(RefCell::new(None), future).await
}
```

Channel code calls `take_last_provider_fallback()` after the request to notify users: "Using Claude Sonnet instead of Opus due to availability."

### Context Truncation

`truncate_for_context()` (lines 288-312) drops oldest half of non-system messages, preserving the most recent user turn:

```rust
fn truncate_for_context(messages: &mut Vec<ChatMessage>) -> usize {
    let non_system: Vec<usize> = messages.iter().enumerate()
        .filter(|(_, m)| m.role != "system")
        .map(|(i, _)| i).collect();
    if non_system.len() <= 1 { return 0; }
    let drop_count = non_system.len() / 2;
    // Remove in reverse order to preserve indices
    for &idx in indices_to_remove.iter().rev() {
        messages.remove(idx);
    }
    drop_count
}
```

Called automatically on context window errors before retry. Bail-fast if nothing to truncate (system prompt alone exceeds context).

### Model Fallback Chains

`with_model_fallbacks()` accepts `HashMap<String, Vec<String>>` mapping primary model to fallback list:

```rust
// Example: "claude-opus" -> ["claude-sonnet", "claude-haiku"]
fn model_chain<'a>(&'a self, model: &'a str) -> Vec<&'a str> {
    let mut chain = vec![model];
    if let Some(fallbacks) = self.model_fallbacks.get(model) {
        chain.extend(fallbacks.iter().map(|s| s.as_str()));
    }
    chain
}
```

### API Key Rotation

`rotate_key()` uses `AtomicUsize.fetch_add()` for round-robin across extra keys on rate limits:

```rust
fn rotate_key(&self) -> Option<&str> {
    if self.api_keys.is_empty() { return None; }
    let idx = self.key_index.fetch_add(1, Ordering::Relaxed) % self.api_keys.len();
    Some(&self.api_keys[idx])
}
```

Note: The warning log at line 512-516 notes the `Provider` trait has no `set_api_key` method, so key rotation cannot actually be applied — it rotates but the key is never used.

### Streaming Support

`stream_chat()` and `stream_chat_with_history()` delegate to the first provider that advertises capability. If tools are requested, only providers reporting `supports_streaming_tool_events()` are considered.

---

## Cross-Cutting Observations

1. **Idempotent schema migrations** in memory — `IF NOT EXISTS` + `schema_sql.contains()` before ALTER
2. **Quote-aware parsing** appears in both security (shell lexer) and memory (FTS5 query construction)
3. **Three-level retry/failover** common pattern: model chain -> provider chain -> retry loop
4. **Error classification granularity**: retryable vs non-retryable vs business logic (plan restrictions)
5. **tokio::task_local** for request-scoped state that must not leak across concurrent requests
6. **SHA-256 truncated** for stable deterministic content keys (avoids `DefaultHasher` instability)
7. **Arc<Mutex<Connection>>** pattern for sharing SQLite across async tasks without connection pooling
