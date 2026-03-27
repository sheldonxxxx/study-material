# Feature Deep Dive: Batch 2 (Features 4-6)

**Date:** 2026-03-26
**Repo:** `/Users/sheldon/Documents/claw/reference/openfang`
**Features:** 16-Layer Security System, Memory & Knowledge Management, REST API & WebSocket Server

---

## Feature 4: 16-Layer Security System

### Overview

Defense-in-depth architecture with 16 independently testable security layers. The implementation spans multiple crates with clear separation between runtime security (sandboxing, audit), type-level security (taint tracking), and protocol security (signing).

### Layer 1-2: WASM Dual-Metered Sandbox (`crates/openfang-runtime/src/sandbox.rs`)

**Architecture:** Uses Wasmtime with dual metering:

1. **Fuel Metering (CPU instructions):** Deterministic, compiler-inserted instruction counting. Allows precise budgets (default 1M fuel units).
2. **Epoch Interruption (wall-clock):** Separate watchdog thread calls `engine.increment_epoch()` after timeout (default 30s). Allows indefinite-running WASM to be terminated.

**Guest ABI Contract:**
```rust
// Exports required from WASM module:
memory: LinearMemory      // Guest linear memory
alloc(size: i32) -> i32   // Allocate in guest memory
execute(input_ptr, input_len) -> i64  // Packed (ptr << 32) | len
```

**Host ABI (imports available to guest):**
- `host_call(request_ptr, request_len) -> i64` - RPC dispatch with capability checks
- `host_log(level, msg_ptr, msg_len)` - Logging without capability checks

**Capability System:** `GuestState` carries `Vec<Capability>` checked before every `host_call` dispatch. Deny-by-default - no filesystem/network unless explicitly granted.

**Clever Pattern - Watchdog Thread Spawn:**
```rust
let engine_clone = engine.clone();
let timeout = config.timeout_secs.unwrap_or(30);
let _watchdog = std::thread::spawn(move || {
    std::thread::sleep(std::time::Duration::from_secs(timeout));
    engine_clone.increment_epoch();
});
```
The watchdog runs on a std thread (not tokio) so it is not affected by async runtime blocking.

**Error Handling:**
- Fuel exhaustion: `Trap::OutOfFuel` caught specifically, returns `SandboxError::FuelExhausted`
- Epoch interrupt: `Trap::Interrupt` caught and converts to human-readable timeout error
- ABI violations: Custom errors for missing exports, out-of-bounds memory access

**Technical Debt / Concerns:**
- Watchdog thread is spawned on every `execute_sync()` call - no pooling or cleanup coordination. Orphaned threads if execution panics (unlikely but possible).
- No memory limit enforcement currently - `max_memory_bytes` field exists in config but is not enforced.
- Guest `alloc` is a simple bump allocator with no free - fine for short-lived executions but could cause memory pressure in long-running scenarios.

### Layer 2: Merkle Hash-Chain Audit Trail (`crates/openfang-runtime/src/audit.rs`)

**Architecture:** Append-only log with SHA-256 chaining. Each entry's hash = SHA256(seq | timestamp | agent_id | action | detail | outcome | prev_hash).

**Genesis Sentinel:** 64 zero characters (`"0".repeat(64)`) for the first entry's prev_hash.

**Persistence:** Optional SQLite backend. On `AuditLog::with_db()`, loads existing entries and verifies chain integrity before returning.

**Verification:** `verify_integrity()` recomputes every hash from genesis and compares against stored hashes. Any mismatch = tampering detected.

**Audit Actions Enum (12 variants):**
```rust
ToolInvoke, CapabilityCheck, AgentSpawn, AgentKill, AgentMessage,
MemoryAccess, FileAccess, NetworkAccess, ShellExec, AuthAttempt,
WireConnect, ConfigChange
```

**Thread Safety:** `Mutex<Vec<AuditEntry>>` and `Mutex<String>` (tip hash) serializes all access.

**Observations:**
- No pruning or compaction strategy - audit log grows indefinitely
- SQLite writes are fire-and-forget (`let _ = conn.execute(...)`) - DB write failures do not fail the audit record, only log a warning
- Hash computation includes all fields as strings - relies on deterministic serialization

### Layer 3: Information Flow Taint Tracking (`crates/openfang-types/src/taint.rs`)

**Model:** Lattice-based taint propagation. Values carry labels; sinks block specific labels.

**Taint Labels (5 variants):**
```rust
ExternalNetwork  // From network requests
UserInput        // Direct user input
Pii              // Personally identifiable
Secret           // API keys, tokens, passwords
UntrustedAgent   // From sandboxed agents
```

**Predefined Sinks:**
- `shell_exec`: Blocks ExternalNetwork, UntrustedAgent, UserInput (prevents injection)
- `net_fetch`: Blocks Secret, Pii (prevents exfiltration)
- `agent_message`: Blocks Secret

**Key Methods:**
- `check_sink()`: Returns `Ok(())` if no blocked labels, `Err(TaintViolation)` otherwise
- `merge_taint()`: Unions label sets when values are concatenated
- `declassify()`: Explicitly removes a label (security decision by caller)

**Observations:**
- Taint labels are HashSet-backed - O(1) lookup in sinks
- No implicit propagation through operations (e.g., string concatenation doesn't automatically merge taint) - caller must explicitly call `merge_taint()`
- Declassification is manual - no automatic sanitization. This is intentional (explicit security decisions) but means misuse is possible.

### Layer 4: Ed25519 Signed Agent Manifests (`crates/openfang-types/src/manifest_signing.rs`)

**Signing Scheme:**
1. SHA-256 hash of manifest content
2. Ed25519 signature over the hash
3. Bundle: manifest + content_hash + signature + signer_public_key + signer_id

**Verification Steps:**
1. Recompute SHA-256 of manifest, compare to stored content_hash
2. Reconstruct public key from signer_public_key bytes
3. Ed25519 signature verification

**SignedManifest Structure:**
```rust
pub manifest: String        // Raw TOML
pub content_hash: String    // Hex SHA-256
pub signature: Vec<u8>      // 64 bytes
pub signer_public_key: Vec<u8>  // 32 bytes
pub signer_id: String       // Human-readable
```

**Observations:**
- Uses `ed25519-dalek` crate - well-audited library
- No key rotation mechanism - signer_id is just a string hint
- No certificate chain or trust model - verification succeeds if signature is valid, doesn't validate signer identity beyond the public key

### Layer 13: Loop Guard (`crates/openfang-runtime/src/loop_guard.rs`)

**Purpose:** Prevents agents from getting stuck in tool call loops.

**Detection Mechanisms:**

1. **Hash-Count Detection:** SHA-256(tool_name | serialized_params) as call identity. Counts identical calls.

2. **Outcome-Aware Detection:** Hashes (tool_call_hash | result_truncated) pairs. Escalates if same call keeps returning identical results.

3. **Ping-Pong Detection:** Sliding window of last 30 calls. Detects A-B-A-B or A-B-C-A-B-C patterns.

4. **Poll Tool Handling:** `shell_exec` with status/poll/watch keywords gets 3x relaxed thresholds (warn at 9 instead of 3, block at 15 instead of 5).

**Graduated Responses:**
- `Allow`: Below thresholds
- `Warn(String)`: Above warn_threshold, message appended to result
- `Block(String)`: Above block_threshold or warning bucket exhausted
- `CircuitBreak(String)`: Global threshold exceeded (30 total calls)

**Backoff Schedule:** Poll tools get progressive delays: 5s -> 10s -> 30s -> 60s (capped).

**Clever Pattern - Warning Bucket:**
```rust
// Prevent warning spam - after max_warnings_per_call, upgrade to Block
let warning_count = self.warnings_emitted.entry(hash.clone()).or_insert(0);
*warning_count += 1;
if *warning_count > self.config.max_warnings_per_call {
    return LoopGuardVerdict::Block(...);
}
```

**Observations:**
- Comprehensive test suite (30+ tests covering all detection modes)
- Uses `serde_json::to_string` for deterministic params serialization (keys always sorted)
- Result truncation to 1000 chars for outcome hashing - avoids hashing massive outputs while still catching identical short results

---

## Feature 5: Memory & Knowledge Management

### Overview

SQLite-backed persistent memory with structured KV, semantic vector search, knowledge graph, and cross-channel session management. Unified behind the `MemorySubstrate` compositing multiple stores.

### Session Management (`crates/openfang-memory/src/session.rs`)

**Dual Session Model:**

1. **Regular Sessions:** Per-channel conversation history. Created per interaction.
2. **Canonical Sessions:** Cross-channel persistent context. One per agent. All channels contribute to it.

**Canonical Session Flow:**
```
User on Telegram -> append_canonical() -> stored in SQLite
User on Discord  -> append_canonical() -> loads canonical + appends
                    -> Agent remembers context from Telegram
```

**Compaction Strategy:**
- Threshold: 100 messages (configurable)
- On exceeding threshold: older messages summarized (truncated text summaries), recent 50 kept
- Summary capped at 4000 chars, UTF-8 safe truncation at char boundaries
- `compaction_cursor` tracks progress for incremental compaction

**JSONL Mirror:**
- Best-effort human-readable export
- Written to `~/.openfang/sessions/{session_id}.jsonl`
- Converts MessageContent blocks to readable format (tool_use, tool_result, thinking blocks all serialized)

**Session Store API:**
```rust
create_session(agent_id) -> Session
save_session(session)
get_session(session_id) -> Option<Session>
delete_session(session_id)
list_sessions() -> Vec<Metadata>
find_session_by_label(agent_id, label) -> Option<Session>
append_canonical(agent_id, messages, threshold) -> CanonicalSession
canonical_context(agent_id, window) -> (Option<summary>, Vec<Message>)
```

### Knowledge Graph (`crates/openfang-memory/src/knowledge.rs`)

**Entities and Relations:**
- `Entity`: id, entity_type (Person, Organization, Project, Custom), name, properties (JSON), timestamps
- `Relation`: source, relation_type (WorksAt, Knows, RelatedTo, etc.), target, confidence (0-1), properties

**Pattern Query:**
```rust
GraphPattern {
    source: Option<String>,      // Entity id or name
    relation: Option<RelationType>,
    target: Option<String>,     // Entity id or name
    max_depth: Option<u32>,    // Currently unused (hardcoded to 1)
}
```

**SQL Construction:** Dynamic query building with boxed trait objects for parameter binding.

**Observations:**
- `query_graph` hardcodes `LIMIT 100` - may need pagination for large graphs
- `max_depth` field exists in GraphPattern but is not used in SQL - only returns direct relations
- `parse_entity` uses `unwrap_or` with defaults on deserialization failure - silently converts invalid data

### Memory Substrate (`crates/openfang-memory/src/substrate.rs`)

**Composition Pattern:**
```rust
struct MemorySubstrate {
    conn: Arc<Mutex<Connection>>,     // Shared SQLite
    structured: StructuredStore,       // KV pairs
    semantic: SemanticStore,           // Vector embeddings
    knowledge: KnowledgeStore,        // Knowledge graph
    sessions: SessionStore,           // Session management
    consolidation: ConsolidationEngine, // Memory decay
    usage: UsageStore,                // Token/cost tracking
}
```

**SQLite Pragmas:**
```rust
conn.execute_batch("PRAGMA journal_mode=WAL; PRAGMA busy_timeout=5000;");
```
- WAL mode: Better concurrent read performance
- busy_timeout: 5s wait on lock contention before erroring

**Async Wrappers:** All blocking SQLite operations wrapped with `tokio::task::spawn_blocking` to avoid blocking the async runtime.

**Task Queue:** Embedded directly in substrate - `task_post`, `task_claim`, `task_complete`, `task_list` with SQLite persistence and priority ordering.

**Observations:**
- `import()` returns stub `ImportReport` with "not yet implemented" error - placeholder for future work
- No vector embedding storage visible in substrate (semantic store exists but embedding generation is elsewhere)
- ConsolidationEngine created but its API not visible in the substrate public methods

---

## Feature 6: REST API & WebSocket Server

### Overview

140+ endpoints built on Axum 0.8. OpenAI-compatible `/v1/chat/completions` drop-in. Serves Alpine.js dashboard SPA. Extensive middleware stack for auth, rate limiting, compression, CORS.

### Server Architecture (`crates/openfang-api/src/server.rs`)

**Middleware Stack (applied in order):**
1. Auth middleware (API key or password hash)
2. GCRA Rate limiter (per-IP token bucket)
3. Security headers (CSP, X-Frame-Options, HSTS, X-Content-Type-Options)
4. Request logging
5. Compression (gzip/brotli)
6. Tracing (tower-http TraceLayer)
7. CORS

**CORS Logic:**
- No auth + `api_key` empty: Permissive localhost origins (127.0.0.1, localhost, plus 3000/8080)
- Auth enabled: Restricted to localhost + configured origins
- Explicit comment noting `CorsLayer::permissive()` is dangerous

**Daemon Lifecycle:**
1. Boot kernel
2. Start background agents
3. Spawn config hot-reload watcher (polls every 30s)
4. Build router with all routes
5. Check for existing daemon (PID + health endpoint verification)
6. Write `daemon.json` with 0600 permissions
7. Start HTTP server with graceful shutdown

**Graceful Shutdown:**
- Unix: SIGINT, SIGTERM, or API notify
- Windows: Ctrl+C or API notify
- Cleans up daemon.json, stops bridges, shuts down kernel

**Key Security Measure - Stale Daemon Detection:**
```rust
if is_process_alive(info.pid) && is_daemon_responding(&info.listen_addr) {
    return Err("Another daemon is already running");
}
// Only removes stale file if process is dead OR not responding
```

### Route Organization (`crates/openfang-api/src/routes.rs`)

**Route Categories:**
- Agent CRUD: `/api/agents`, `/{id}/...`
- Sessions: `/api/sessions`, `/{id}/sessions/...`
- Memory: `/api/memory/agents/{id}/kv/...`
- Channels: `/api/channels/...`
- Skills/Hands: `/api/skills`, `/api/hands`
- Budget/Usage: `/api/budget`, `/api/usage`
- Triggers/Workflows/Schedules
- MCP: `/mcp`, `/api/mcp/servers`
- A2A: `/.well-known/agent.json`, `/a2a/...`
- OpenAI compat: `/v1/chat/completions`, `/v1/models`
- Auth: `/api/auth/login`, `/api/auth/logout`, `/api/auth/check`

**Security-Critical Spawn Path (`spawn_agent`):**
1. Template name sanitization (alphanumeric, dash, underscore only)
2. Path traversal prevention (UUID validation for file_id)
3. 1MB manifest size limit
4. Ed25519 manifest signature verification (when signed_manifest provided)
5. Audit log record on signature failure

**Attachment Resolution:**
- File IDs must be valid UUIDs (prevents path traversal)
- Content type must start with `image/` to be processed
- Files read from temp dir `openfang_uploads`
- Base64-encoded into ContentBlock::Image

### OpenAI Compatibility (`crates/openfang-api/src/openai_compat.rs`)

**Agent Resolution:**
1. `openfang:{name}` -> registry.find_by_name
2. Valid UUID -> registry.get
3. Plain string -> registry.find_by_name (fallback)

**Message Conversion:**
- OpenAI content can be string, array of parts (text/image_url), or null
- Image URLs parsed as data URIs: `data:{media_type};base64,{data}`
- Non-data-URL images not supported (would require fetching)

**Streaming Response:**
- SSE with `text/event-stream` content type
- Initial role delta sent before any content
- Tool calls streamed as `{index, id, type, function{name, arguments}}` chunks
- `ContentComplete` with `StopReason::ToolUse` does NOT finish - waits for tool result
- `ContentComplete` with `EndTurn/MaxTokens/StopSequence` sends finish

**Non-Streaming Response:**
- Single `ChatCompletionResponse` with full text
- Usage info from kernel's token accounting

**Observations:**
- No `functions` tool calling mode support (only `function_call` / `function` parts in content)
- No `stop` parameter support for controlling generation
- No `presence_penalty` / `frequency_penalty` parameters
- `max_tokens` accepted but may not be propagated to all LLM drivers

---

## Cross-Feature Observations

### Security Integration
- Manifest signing (Feature 4) is enforced at spawn time (Feature 6 route)
- Taint tracking (Feature 4) is applied in tool execution (Feature 2)
- Audit log (Feature 4) records agent operations from across the kernel

### Memory Integration
- Sessions (Feature 5) feed into agent context windows (Feature 2)
- Knowledge graph (Feature 5) populated by agent tool results
- Substrate (Feature 5) used by API routes (Feature 6) for KV access

### Design Patterns

**1. Arc<Mutex<Connection>> for SQLite:**
```rust
conn: Arc<Mutex<Connection>>
```
Used across session, knowledge, and semantic stores. Allows cloned handles to share the same connection with serialized access.

**2. spawn_blocking for All SQLite Operations:**
```rust
tokio::task::spawn_blocking(move || {
    store.operation()
})
.await?
```
Prevents blocking the async runtime. Every substrate method follows this pattern.

**3. Option<Arc<T>> for Optional Components:**
```rust
pub peer_registry: Option<Arc<PeerRegistry>>
```
Components can be absent without null-checking throughout the codebase.

**4. Hex-Encoded Hashes for Audit Trail:**
- SHA-256 outputs bytes, stored as hex string
- Enables easy debugging and JSON serialization
- Consistent 64-character length for zero-values

### Potential Issues

1. **Audit Log Growth:** No pruning strategy - audit trail grows indefinitely with no cleanup.

2. **Taint Declassification Responsibility:** Manual `declassify()` calls mean a careless caller can forget to sanitize tainted data before sensitive sinks.

3. **Watchdog Thread Spawn Per Execution:** In sandbox.rs, every `execute_sync()` spawns a new watchdog thread. If execution panics, the thread is orphaned (harmless but wasteful).

4. **GCRA Rate Limiter on `Arc<GcraLimiter>`:** The limiter is shared globally - no per-tenant or per-endpoint granularity visible in the middleware setup.

5. **OpenAI Compat Missing Features:** No `functions` tool calling, no `stop` parameter, no penalty parameters. Limits compatibility with some clients.

6. **Import Not Implemented:** `Memory::import()` returns a stub error - data import is explicitly deferred.

7. **Knowledge Graph max_depth Unused:** The field exists in GraphPattern but the SQL query doesn't use it - always returns depth-1 relations only.
