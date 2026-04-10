# Feature Deep Dive: Batch 5 (Features 13-14)

**Features:** Migration Engine, WhatsApp Web Gateway
**Research:** 2026-03-26
**Files Analyzed:**
- `crates/openfang-migrate/src/lib.rs`
- `crates/openfang-migrate/src/openclaw.rs` (3200+ lines)
- `crates/openfang-migrate/src/report.rs`
- `crates/openfang-migrate/Cargo.toml`
- `crates/openfang-kernel/src/whatsapp_gateway.rs`
- `packages/whatsapp-gateway/index.js`
- `packages/whatsapp-gateway/package.json`

---

## Feature 13: Migration Engine

### Purpose

One-command migration from OpenClaw (and designed for LangChain/AutoGPT future support) with automatic import of agents, memory, skills, sessions, channel configs, and credentials.

### Architecture

```
MigrateOptions
    ├── source: MigrateSource (OpenClaw | LangChain | AutoGpt)
    ├── source_dir: PathBuf
    ├── target_dir: PathBuf
    └── dry_run: bool
         │
         ▼
run_migration() [lib.rs]
    │
    ├── OpenClaw ──────────────────────────────────────┐
    │    │                                             │
    │    ▼                                             │
    │ migrate() [openclaw.rs]                          │
    │    │                                             │
    │    ├─ detect_openclaw_home()                     │
    │    │   (OPENCLAW_STATE_DIR env, ~/.openclaw,     │
    │    │    ~/.clawdbot, ~/.moldbot, ~/.moltbot,      │
    │    │    Windows APPDATA)                          │
    │    │                                             │
    │    ├─ find_config_file()                         │
    │    │   (prefers JSON5: openclaw.json >           │
    │    │    clawdbot.json > moldbot.json >            │
    │    │    moltbot.json, falls back to config.yaml) │
    │    │                                             │
    │    ├─ migrate_from_json5() or migrate_from_legacy_yaml()
    │    │   ├─ migrate_config_from_json()
    │    │   │   → writes config.toml
    │    │   │   → extracts default_model, channels
    │    │   │   → writes secrets.env
    │    │   │
    │    │   ├─ migrate_agents_from_json()
    │    │   │   → converts agent entries to agent.toml
    │    │   │   → maps provider names
    │    │   │   → maps tools via openfang_types
    │    │   │
    │    │   ├─ migrate_memory_files()
    │    │   │   → copies memory/{agent}/MEMORY.md
    │    │   │   → copies agents/{agent}/MEMORY.md (legacy)
    │    │   │
    │    │   ├─ migrate_workspace_dirs()
    │    │   │   → copies workspaces/{agent}/ contents
    │    │   │   → copies agents/{agent}/workspace/ (legacy)
    │    │   │
    │    │   ├─ migrate_sessions()
    │    │   │   → copies .jsonl files to imported_sessions/
    │    │   │
    │    │   └─ report_skipped_features()
    │    │       (auth-profiles, cron, hooks, vector index, etc.)
    │    │
    │    └─ MigrationReport.to_markdown()
    │
    └── report.rs ── MigrationReport
         ├── imported: Vec<MigrateItem>
         ├── skipped: Vec<SkippedItem>
         ├── warnings: Vec<String>
         ├── to_markdown() → String
         └── print_summary()
```

### Supported Migration Items

| Item | Source Layout | Target | Notes |
|------|--------------|--------|-------|
| Config | `openclaw.json` / `config.yaml` | `config.toml` | TOML serialization |
| Agents | `agents.list[]` (JSON5) / `agents/{name}/agent.yaml` | `agents/{name}/agent.toml` | Full manifest conversion |
| Memory | `memory/{agent}/MEMORY.md` | `agents/{agent}/imported_memory.md` | Both modern and legacy layouts |
| Workspaces | `workspaces/{agent}/` | `agents/{agent}/workspace/` | Directory tree copy |
| Sessions | `sessions/*.jsonl` | `imported_sessions/*.jsonl` | JSONL conversation logs |
| Channel Tokens | `channels.{name}.*` | `secrets.env` | Bot tokens written with 0o600 perms |
| WhatsApp Creds | `channels.whatsapp.auth_dir/` | `credentials/whatsapp/` | Baileys auth dir copy |

### Channel Support (13 Channels)

| Channel | Config Field | Secret | Status |
|---------|-------------|--------|--------|
| Telegram | `bot_token_env` | `TELEGRAM_BOT_TOKEN` | Supported |
| Discord | `bot_token_env` | `DISCORD_BOT_TOKEN` | Supported |
| Slack | `bot_token_env`, `app_token_env` | `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN` | Supported |
| WhatsApp | `auth_dir` + `access_token_env` | `WHATSAPP_ACCESS_TOKEN` | Partial (Baileys creds copied) |
| Signal | `http_url` / `http_host` + `http_port` | None | Supported |
| Matrix | `access_token` | `MATRIX_ACCESS_TOKEN` | Supported |
| Google Chat | `service_account_file` | `GOOGLE_CHAT_SA_FILE` | Supported |
| Teams | `app_id`, `app_password` | `TEAMS_APP_PASSWORD` | Supported |
| IRC | `host`, `port`, `nick`, `password` | `IRC_PASSWORD` | Supported |
| Mattermost | `bot_token`, `base_url` | `MATTERMOST_TOKEN` | Supported |
| Feishu | `app_id`, `app_secret`, `domain` | `FEISHU_APP_SECRET` | Supported |
| iMessage | — | — | Skipped (macOS-only) |
| BlueBubbles | — | — | Skipped (no adapter) |

### Provider Mapping (20+ Providers)

```rust
fn map_provider(openclaw_provider: &str) -> String {
    match openclaw_provider.to_lowercase().as_str() {
        "anthropic" | "claude" => "anthropic",
        "openai" | "gpt" => "openai",
        "groq" | "ollama" | "openrouter" | "deepseek" | "together" | "mistral"
        | "fireworks" | "google" | "gemini" | "qwen" | "dashscope" | "moonshot"
        | "kimi" | "minimax" | "zhipu" | "glm" | "qianfan" | "baidu" | "xai"
        | "grok" | "cerebras" | "sambanova" | "perplexity" | "cohere" | "ai21"
        | "huggingface" | "replicate" | "github-copilot" | "copilot" | "vllm"
        | "lmstudio" => canonical_name,
        other => other.to_string(),
    }
}
```

Also maps API key env vars: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GROQ_API_KEY`, etc.

### Skipped Items (with reasons)

| Item | Reason |
|------|--------|
| `auth-profiles.json` | Security: credentials must be set as env vars |
| `memory-search/index.db` | SQLite vector index not portable |
| `cron-store.json` | Cron run state not portable |
| `hooks` | Different event system in OpenFang |
| iMessage channel | macOS-only, requires manual setup |
| BlueBubbles channel | No OpenFang adapter |
| Skills | Must be reinstalled via `openfang skill install` |
| LangChain / AutoGPT | Not yet implemented |

### Key Technical Decisions

1. **Dry-run by default for secrets**: The migration warns about credentials and gives users a chance to review before writing
2. **Content-hash change detection**: `write_if_changed()` uses FNV hash to avoid unnecessary file writes and npm reinstalls
3. **Dual-layout memory support**: Checks both `memory/{agent}/MEMORY.md` and `agents/{agent}/MEMORY.md`
4. **Tool compatibility delegation**: `is_known_openfang_tool()` and `map_tool_name()` live in `openfang_types::tool_compat` so migration and kernel share identical definitions
5. **Policy normalization**: `map_dm_policy()` and `map_group_policy()` translate OpenClaw policies to OpenFang equivalents

### Technical Debt / Shortcuts

1. **WhatsApp re-auth warning**: When migrating WhatsApp config, the code copies the Baileys auth_dir but warns that re-authentication may be needed
2. **Silent tool skipping**: Tools without OpenFang equivalents are silently skipped with warnings rather than failing migration
3. **Agent ID as name fallback**: If `name` field missing, uses `id` as the agent name
4. **Provider inference from model ref**: If model ref contains `/` (e.g., `anthropic/claude-sonnet-4`), splits to get provider and model

### Code Statistics

- `openclaw.rs`: ~3300 lines (largest single file in crate)
- `report.rs`: ~210 lines (with tests)
- `lib.rs`: ~77 lines (framework entry point)

---

## Feature 14: WhatsApp Web Gateway

### Purpose

QR-code-based WhatsApp connection (like WhatsApp Web) that enables WhatsApp messaging without requiring a Meta Business account. Implemented as a standalone Node.js service managed by the Rust kernel.

### Architecture

```
OpenFang Kernel (Rust)
    │
    ├─ start_whatsapp_gateway() [whatsapp_gateway.rs]
    │   ├─ node_available() ─── checks for Node.js >= 18
    │   ├─ ensure_gateway_installed()
    │   │   ├─ write_if_changed(index.js) [from include_str!]
    │   │   ├─ write_if_changed(package.json)
    │   │   └─ npm install --production (if needed)
    │   │
    │   └─ spawn node index.js as child process
    │       ├─ WHATSAPP_GATEWAY_PORT=3009
    │       ├─ OPENFANG_URL=http://127.0.0.1:4200
    │       └─ OPENFANG_DEFAULT_AGENT=assistant
    │
    └─ crash monitoring loop (5s/10s/20s backoff, max 3 restarts)

packages/whatsapp-gateway/index.js (Node.js)
    │
    ├─ Baileys socket (makeWASocket)
    │   ├─ useMultiFileAuthState('./auth_store')
    │   └─ printQRInTerminal: true
    │
    ├─ Connection state machine
    │   ├─ QR code → convert to data:image/png;base64 via qrcode
    │   ├─ connected → clear QR, track session
    │   ├─ disconnected (loggedOut) → delete auth_store, stop
    │   └─ disconnected (other) → exponential backoff reconnect
    │
    ├─ messages.upsert handler
    │   ├─ Extract text from conversation | extendedTextMessage | imageMessage
    │   ├─ forwardToOpenFang() → POST /api/agents/{agent}/message
    │   └─ Reply via sock.sendMessage()
    │
    └─ HTTP server (built-in http module)
        ├─ POST /login/start → start Baileys, return QR
        ├─ GET /login/status → connection status
        ├─ POST /message/send → outgoing message
        └─ GET /health → health check
```

### Dependencies

```json
{
  "@whiskeysockets/baileys": "^6",  // WhatsApp Web protocol
  "qrcode": "^1.5",                  // QR code generation
  "pino": "^9"                       // Structured logging
}
```

### HTTP API

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/login/start` | POST | Start Baileys connection, return QR as data URL |
| `/login/status` | GET | Poll connection status (connected/qr_ready/disconnected) |
| `/message/send` | POST | Send outgoing WhatsApp message |
| `/health` | GET | Gateway health check |

### Message Flow

```
[WhatsApp User] ──sends message──▶ [WhatsApp Mobile App]
                                         │
                                         ▼
                              [Baileys Web Socket (index.js)]
                                         │
                                         │ messages.upsert event
                                         │ Extracts: sender JID, text, pushName
                                         ▼
                              [POST /api/agents/{agent}/message]
                                         │
                                         ▼ (async HTTP)
                              [OpenFang Kernel / LLM]
                                         │
                                         ▼ (response text)
                              [sock.sendMessage(replyJid, { text })]
                                         │
                                         ▼
                              [WhatsApp User] ──receives reply──▶
```

### Auth State Persistence

Baileys stores auth in `./auth_store/`:
- `creds.json` — session credentials (decrypted)
- `app-state-sync.json` — message sync state
- `pre-keys.json` — key material

The gateway checks for existing `creds.json` on startup and auto-connects if found, avoiding the need to re-scan QR on every restart.

### Reconnection Logic

```javascript
// Exponential backoff: 1s, 2s, 4s, 8s... capped at 60s
const delay = Math.min(1000 * Math.pow(2, reconnectAttempt - 1), MAX_RECONNECT_DELAY);
setTimeout(() => startConnection(), delay);

// Non-recoverable: loggedOut from phone
if (statusCode === DisconnectReason.loggedOut) {
    // Delete auth_store so next start gets fresh QR
    fs.rmSync(authPath, { recursive: true, force: true });
}
```

### Clever Design Patterns

1. **QR as Data URL**: QR codes converted to `data:image/png;base64,...` format for easy polling via HTTP GET. Avoids WebSocket complexity.

2. **Kernel-embedded JS**: The gateway JS is embedded at compile time via `include_str!("../../../packages/whatsapp-gateway/index.js")`, so the gateway travels with the binary.

3. **Content-hash updates**: `write_if_changed()` with FNV hash avoids unnecessary `npm install` when only JS code changes.

4. **PID tracking**: Kernel stores gateway PID in `whatsapp_gateway_pid: Arc<Mutex<Option<u32>>>` for clean shutdown.

5. **Weak kernel reference**: Uses `Arc::downgrade()` to detect if kernel is shutting down without preventing process exit.

6. **Graceful shutdown**: SIGINT/SIGTERM handlers ensure `sock.end()` is called before exit.

### Limitations / Technical Debt

1. **Text-only messages**: While the code detects media types (images, videos, stickers, documents), it only forwards text content to OpenFang. Media files are not uploaded or forwarded.

2. **Single agent routing**: All WhatsApp messages go to `DEFAULT_AGENT`. No per-sender or per-group routing logic.

3. **No message threading**: Replies use simple JID construction; no conversation threading awareness.

4. **HTTP polling for QR**: No WebSocket or SSE for real-time QR delivery. Clients must poll `/login/status`.

5. **Node.js as external process**: The Rust kernel must shell out to Node.js, adding process management complexity. Not a true Rust-native solution.

6. **No group admin awareness**: Group messages include `group_jid` in metadata but no group configuration or admin features.

### Code Statistics

- `whatsapp_gateway.rs`: ~344 lines (with 5 tests)
- `index.js`: ~388 lines
- `package.json`: 23 lines

---

## Cross-Feature Observations

1. **Shared tool compatibility**: Both features reference `openfang_types::tool_compat` for tool name mapping, ensuring consistent behavior between migration and runtime.

2. **WhatsApp bridging**: The migration engine handles WhatsApp token/secrets migration, while the gateway handles actual WhatsApp connectivity. These are complementary but separate systems.

3. **No CI/testing for migrations**: The `openclaw.rs` has inline tests with `tempfile::TempDir` but no integration tests against real OpenClaw workspaces.

4. **Documentation in code**: Extensive module-level documentation (`//!`) explains the OpenClaw directory layout and migration decisions, making the codebase self-documenting.
