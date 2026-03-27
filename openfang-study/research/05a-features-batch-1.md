# Features 1-3 Deep Dive: Batch Analysis

**Project:** OpenFang (Rust Agent OS)
**Analysis Date:** 2026-03-26
**Source Files Examined:** `crates/openfang-hands/src/{lib,registry,bundled}.rs`, `crates/openfang-runtime/src/{agent_loop,tool_runner}.rs`, `crates/openfang-runtime/src/drivers/mod.rs`, `crates/openfang-channels/src/{bridge,lib,types,discord}.rs`

---

## Feature 1: Autonomous "Hands" System

### What It Is

A Hand is a pre-built, domain-complete agent configuration that runs autonomously on a schedule without user prompting. Unlike regular agents (you chat with them), Hands work for you (you check in on them). The system provides 8 bundled Hands with complete pipelines (Clip, Lead, Collector, Predictor, Researcher, Twitter, Browser, Trader).

### Architecture

**Core Files:**
- `crates/openfang-hands/src/lib.rs` (866 lines) - Type definitions
- `crates/openfang-hands/src/registry.rs` (903 lines) - HandRegistry with concurrent storage
- `crates/openfang-hands/src/bundled.rs` (200+ lines) - Compile-time hand loading
- `crates/openfang-hands/bundled/{clip,lead,collector,predictor,researcher,twitter,browser,trader}/` - 8 bundled hand definitions

**Key Types:**
```rust
HandDefinition      // Parsed from HAND.toml
HandInstance        // Runtime instance linking definition to agent
HandStatus          // Active | Paused | Error(String) | Inactive
HandRequirement     // Binary | EnvVar | ApiKey with install instructions
HandSetting         // Select | Text | Toggle with options
HandCategory        // Content | Security | Productivity | Development | etc.
```

### Code Flow

```
HandRegistry::load_bundled()
  └── bundled_hands() returns Vec of (id, HAND.toml, SKILL.md) via include_str!()
        └── parse_bundled() → HandDefinition with skill_content attached

HandRegistry::activate(hand_id, config)
  └── Creates HandInstance (not spawning agent — kernel handles that)
  └── set_agent() called by kernel after spawning

HandRegistry::readiness(hand_id)
  └── cross-references requirements + active instances
  └── Returns HandReadiness { requirements_met, active, degraded }

resolve_settings(settings, config)
  └── Generates prompt_block (markdown summary)
  └── Collects env_vars from selected options' provider_env fields
```

### Hands TOML Structure (from `clip/HAND.toml` and `lead/HAND.toml`)

```toml
id = "clip"
name = "Clip Hand"
description = "Turns long-form video into viral short clips"
category = "content"
tools = ["shell_exec", "file_read", "file_write", "file_list", ...]

# Requirements with platform-specific install instructions
[[requires]]
key = "ffmpeg"
label = "FFmpeg must be installed"
requirement_type = "binary"
check_value = "ffmpeg"

[requires.install]
macos = "brew install ffmpeg"
windows = "winget install Gyan.FFmpeg"
linux_apt = "sudo apt install ffmpeg"
manual_url = "https://ffmpeg.org/download.html"
estimated_time = "2-5 min"

# Configurable settings with provider_env and binary availability
[[settings]]
key = "stt_provider"
label = "Speech-to-Text Provider"
setting_type = "select"
default = "auto"

[[settings.options]]
value = "groq_whisper"
label = "Groq Whisper API"
provider_env = "GROQ_API_KEY"    # checked for "Ready" badge

[[settings.options]]
value = "whisper_local"
label = "Local Whisper"
binary = "whisper"               # checked on PATH

[agent]
name = "clip-hand"
provider = "default"
model = "default"
max_tokens = 8192
temperature = 0.4
system_prompt = """You are Clip Hand — ..."""

[dashboard]
[[dashboard.metrics]]
label = "Clips Generated"
memory_key = "clip_hand_clips_generated"
format = "number"
```

### Requirements Checking (registry.rs lines 428-636)

**Binary Check:**
```rust
fn check_requirement(req: &HandRequirement) -> bool {
    match req.requirement_type {
        RequirementType::Binary => {
            // Special case: python3/python — actually runs the command
            // to verify output contains "Python 3" (Windows Store shim avoidance)
            if req.check_value == "python3" || req.check_value == "python" {
                return check_python3_available();
            }
            which_binary(&req.check_value)  // PATH scan with Windows .exe/.cmd/.bat
        }
        RequirementType::EnvVar | RequirementType::ApiKey => {
            std::env::var(&req.check_value).map(|v| !v.is_empty()).unwrap_or(false)
        }
    }
}
```

**Python3 Detection (lines 463-495):**
```rust
fn check_python3_available() -> bool {
    run_returns_python3("python3") || run_returns_python3("python")
}
fn run_returns_python3(cmd: &str) -> bool {
    // Actually runs: python3 --version (or python --version)
    // Checks stdout AND stderr for "Python 3"
    // Avoids Windows Store shim that just opens Microsoft Store
}
```

**Chromium Detection (lines 504-584):**
Checks in order:
1. `CHROME_PATH` / `CHROMIUM_PATH` env vars
2. Common binary names on PATH (chromium, chromium-browser, google-chrome, etc.)
3. Well-known install paths (Windows Program Files, macOS Applications, Linux /usr/bin)
4. Playwright cache (`~/.cache/ms-playwright/chromium-*`)

### Clever Solutions

**1. Dual TOML Format Support (lib.rs lines 319-326):**
```rust
pub fn parse_hand_toml(content: &str) -> Result<HandDefinition, toml::de::Error> {
    // Try flat format first
    if let Ok(def) = toml::from_str::<HandDefinition>(content) {
        return Ok(def);
    }
    // Fall back to [hand] wrapper table format
    let wrapper: HandTomlWrapper = toml::from_str(content)?;
    Ok(wrapper.hand)
}
```

**2. Issue #402 - Cron Reassignment After Restart (registry.rs lines 76-106):**
`load_state()` returns `(hand_id, config, old_agent_id)` so kernel can reassign cron jobs to new agent after restart.

**3. Degraded State for Optional Requirements (registry.rs lines 376-405):**
```rust
let degraded = active && reqs.iter().any(|(_, ok)| !ok);
```
A hand is "degraded" (not fully functional) if it's active but has unmet optional requirements (e.g., browser hand without optional chromium).

**4. Settings Availability Checking (registry.rs lines 612-636):**
```rust
fn check_option_available(provider_env, binary) -> bool {
    // GEMINI_API_KEY special case: also accepts GOOGLE_API_KEY
    // If provider_env set, binary map is checked too
}
```

### Technical Debt / Concerns

1. **No Hand uninstall operation** - Only install (from path or content), no explicit uninstall. Overwriting via `upsert_from_content` works but is implicit.

2. **`bundled_hands()` hardcodes 8 hands** - Adding a new bundled hand requires code changes (adding another `include_str!` line). No dynamic discovery.

3. **SKILL.md content embedded at compile time** - If skill content needs to be updated, must recompile. However, `skill_content: Option<String>` suggests runtime loading is also supported.

---

## Feature 2: Agent Runtime Engine

### What It Is

The core agent execution loop that orchestrates LLM calls, tool execution, memory management, and safety systems. The agent loop handles receiving a user message, recalling relevant memories, calling the LLM, executing tool calls, and saving the conversation.

### Architecture

**Core Files:**
- `crates/openfang-runtime/src/agent_loop.rs` (183KB, 1100+ lines) - Main loop
- `crates/openfang-runtime/src/tool_runner.rs` (155KB) - Tool execution
- `crates/openfang-runtime/src/drivers/mod.rs` (400+ lines) - Driver factory
- `crates/openfang-runtime/src/llm_driver.rs` (150+ lines) - Trait definition
- `crates/openfang-runtime/src/drivers/{anthropic,gemini,openai,copilot,claude_code,qwen_code,fallback}.rs` - Driver implementations

### Agent Loop Flow (agent_loop.rs lines 145-917)

```
run_agent_loop(manifest, user_message, session, memory, driver, ...)
  │
  ├─ 1. Memory recall (vector embedding or text fallback)
  │
  ├─ 2. System prompt build (base + recalled memories)
  │
  ├─ 3. Add user message to session history (multimodal if blocks provided)
  │
  ├─ 4. History emergency trim (if > 20 messages)
  │
  ├─ 5. FOR each iteration (max_iterations, default 50):
  │     │
  │     ├─ Context overflow recovery (7-phase pipeline)
  │     │
  │     ├─ Context guard (compact oversized tool results)
  │     │
  │     ├─ LLM call_with_retry():
  │     │     - Provider cooldown circuit breaker
  │     │     - Exponential backoff (rate limited / overloaded)
  │     │     - Up to 3 retries
  │     │     - ModelNotFound fallback chain
  │     │     - Error classification via llm_errors::classify_error()
  │     │
  │     ├─ Handle stop_reason:
  │     │     - EndTurn | StopSequence → extract text, save session
  │     │     - ToolUse → execute tools
  │     │     - MaxTokens → continue with "Please continue."
  │     │
  │     └─ Tool execution (lines 629-786):
  │           - Loop guard check (circuit breaker)
  │           - BeforeToolCall hook (can block)
  │           - 120 second timeout
  │           - Dynamic truncation based on context budget
  │           - AfterToolCall hook
  │           - Interim session save (data loss prevention)
  │
  └─ 6. Return AgentLoopResult { response, total_usage, iterations, cost_usd, silent, directives }
```

### Key Safeguard Systems

**1. Loop Guard - Circuit Breaker (lines 631-667):**
```rust
let verdict = loop_guard.check(&tool_call.name, &tool_call.input);
match verdict {
    LoopGuardVerdict::CircuitBreak(msg) => {
        // Save session, fire hook, return error
        return Err(OpenFangError::Internal(msg.clone()));
    }
    LoopGuardVerdict::Block(msg) => {
        // Return error as tool result, continue to next tool
        tool_result_blocks.push(ToolResult { is_error: true, ... });
        continue;
    }
    LoopGuardVerdict::Warn(msg) => {
        // Append warning to tool result
        final_content = format!("{content}\n\n[LOOP GUARD] {warn_msg}");
    }
    LoopGuardVerdict::Allow => { /* proceed */ }
}
```

**2. Phantom Action Detection (lines 56-66, 498-513):**
Detects when LLM claims to have sent/posted/emailed without calling the corresponding tool:
```rust
fn phantom_action_detected(text: &str) -> bool {
    let has_action = ["sent ", "posted ", "emailed "].iter().any(|v| lower.contains(v));
    let has_channel = ["telegram", "whatsapp", "slack", ...].iter().any(|c| lower.contains(c));
    has_action && has_channel
}
```
If detected, re-prompts with system guidance forcing real tool usage.

**3. Empty Response Retry (lines 457-478):**
One-shot retry on empty response (iteration 0 or silent failure). Re-validates message history before retry.

**4. Tool Error Guidance (lines 69-82, 813-830):**
After failed tool calls, injects system prompt guidance:
```
"[System: One or more tool calls failed. Do NOT invent missing results,
cite nonexistent search results, or pretend failed tools succeeded.]"
```

**5. Context Overflow Recovery (line 350-361):**
Replaces old `emergency_trim_messages`. Uses `recover_from_overflow()` 7-phase pipeline. Re-validates tool_call/tool_result pairing after drains.

**6. Session Repair (line 289, 319, 360, 471):**
Validates and repairs message history throughout the loop - drops orphans, merges consecutive same-role messages.

**7. Interim Session Save (lines 840-843):**
```rust
if let Err(e) = memory.save_session_async(session).await {
    warn!("Failed to interim-save session: {e}");
}
```
Saves after tool execution to prevent data loss on crash.

### LLM Driver System

**Trait (llm_driver.rs lines 145+):**
```rust
#[async_trait]
pub trait LlmDriver: Send + Sync {
    async fn complete(&self, request: CompletionRequest) -> Result<CompletionResponse, LlmError>;
    async fn stream(&self, request: CompletionRequest, tx: mpsc::Sender<StreamEvent>) -> Result<CompletionResponse, LlmError>;
}
```

**Driver Factory (drivers/mod.rs lines 257-400+):**
```rust
pub fn create_driver(config: &DriverConfig) -> Result<Arc<dyn LlmDriver>, LlmError> {
    match provider {
        "anthropic" => Arc::new(anthropic::AnthropicDriver::new(api_key, base_url)),
        "gemini" | "google" => Arc::new(gemini::GeminiDriver::new(api_key, base_url)),
        "openai" | "groq" | "deepseek" | ... => Arc::new(openai::OpenAIDriver::new(api_key, base_url)),
        "claude-code" => Arc::new(claude_code::ClaudeCodeDriver::new(cli_path, skip_permissions)),
        "qwen-code" => Arc::new(qwen_code::QwenCodeDriver::new(cli_path, skip_permissions)),
        "github-copilot" | "copilot" => Arc::new(copilot::CopilotDriver::new(github_token, base_url)),
        "azure" | "azure-openai" => Arc::new(openai::OpenAIDriver::new_azure(api_key, base_url)),
        "kimi_coding" => Arc::new(anthropic::AnthropicDriver::new(api_key, base_url)), // Anthropic-compatible
        _ => { /* OpenAI-compatible fallback */ }
    }
}
```

**27+ Providers via OpenAI-compatible format:**
groq, openrouter, deepseek, together, mistral, fireworks, ollama, vllm, lmstudio, perplexity, cohere, ai21, cerebras, sambanova, huggingface, xai, replicate, chutes, venice, nvidia-nim, azure-openai, volcengine, volcengine_coding, kimi_coding, lemonade, qianfan, moonshot, zhipu, zhipu_coding, zai, zai_coding, and any custom provider with base_url.

### Tool Execution (tool_runner.rs lines 99-526)

Single `execute_tool()` function with pattern matching:
```rust
pub async fn execute_tool(...) -> ToolResult {
    let tool_name = normalize_tool_name(tool_name);  // LLM alias resolution

    // Capability enforcement
    if let Some(allowed) = allowed_tools {
        if !allowed.iter().any(|t| t == tool_name) {
            return ToolResult { is_error: true, content: "Permission denied" };
        }
    }

    // Approval gate
    if kh.requires_approval(tool_name) {
        match kh.request_approval(...).await {
            Ok(true) => { /* proceed */ }
            Ok(false) => { return ToolResult { is_error: true, content: "Denied" }; }
            Err(e) => { return ToolResult { is_error: true, content: "Approval error" }; }
        }
    }

    // Execute via match
    match tool_name {
        "file_read" => tool_file_read(input, workspace_root).await,
        "file_write" => tool_file_write(input, workspace_root).await,
        "shell_exec" => { /* metachar check + exec policy + taint check */ },
        "web_fetch" => { /* taint check + fetch */ },
        "browser_navigate" => { /* taint check + browser tool */ },
        // ... 50+ tools total
        other => {
            // MCP tool fallback: mcp_{server}_{tool}
            // Skill registry fallback
            // Unknown tool error
        }
    }
}
```

**Shell Exec Security (lines 214-266):**
1. Shell metacharacter check (ALWAYS runs, even in Full mode)
2. Exec policy validation (allowlist/deny/full)
3. Heuristic taint check (SKIP for Full exec policy):
   ```rust
   if !is_full_exec {
       if let Some(violation) = check_taint_shell_exec(command) {
           return ToolResult { is_error: true, content: "Taint violation" };
       }
   }
   ```

**Web Fetch Taint Check (lines 54-75):**
Blocks URLs containing `api_key=`, `token=`, `secret=`, `password=` in query params (data exfiltration).

### Clever Patterns

**1. ModelNotFound Fallback Chain (lines 1020-1085):**
When primary model returns ModelNotFound error, tries fallback models in order before propagating error. Creates new driver for each fallback.

**2. Provider Prefix Stripping (lines 88-98):**
```rust
pub fn strip_provider_prefix(model: &str, provider: &str) -> String {
    // "openrouter/google/gemini-2.5-flash" → "google/gemini-2.5-flash"
    // Removes provider prefix so upstream API receives expected format
}
```

**3. Phantom Action Detection:**
Detects LLM claiming to have performed channel actions (sent, posted, emailed) without calling tools. Forces re-prompt for real tool usage.

**4. Dynamic Truncation:**
```rust
let content = truncate_tool_result_dynamic(&result.content, &context_budget);
```
Context-aware tool result truncation (not flat MAX_CHARS) based on remaining context budget.

**5. Message Strip for Multimodal:**
```rust
// Strip Image blocks from session to prevent base64 bloat
// LLM already received them via llm_messages above
for msg in session.messages.iter_mut() {
    if let MessageContent::Blocks(blocks) = &mut msg.content {
        let had_images = blocks.iter().any(|b| matches!(b, ContentBlock::Image { .. }));
        if had_images {
            blocks.retain(|b| !matches!(b, ContentBlock::Image { .. }));
            if blocks.is_empty() {
                blocks.push(ContentBlock::Text { text: "[Image processed]".to_string(), ... });
            }
        }
    }
}
```

**6. NO_REPLY / Silent Directive:**
Agent can return `NO_REPLY` or set `silent: true` in directives to intentionally not respond.

### Technical Debt / Concerns

1. **`call_with_retry()` creates new fallback drivers per attempt (line 1041):**
   ```rust
   let fb_driver = match crate::drivers::create_driver(&fb_config) {
   ```
   Each fallback model creates a fresh driver instance. This is fine for infrequent fallback use but adds overhead.

2. **No cancellation support in the loop** - Once LLM call starts, can't be cancelled except via timeout. No graceful interrupt mechanism.

3. **`Phantom action detection is naive` - Simple keyword matching could produce false positives/negatives.

4. **MAX_ITERATIONS default 50 may be too high or low** - Made configurable per agent via `manifest.autonomous.max_iterations`.

---

## Feature 3: Multi-Channel Messaging (40 Adapters)

### What It Is

Connect agents to every platform users are on. 50+ pluggable channel adapters that convert platform-specific messages into unified `ChannelMessage` events for the kernel. Each adapter supports per-channel model overrides, DM/group policies, rate limiting, and output formatting.

### Architecture

**Core Files:**
- `crates/openfang-channels/src/bridge.rs` (500+ lines) - BridgeManager, dispatch logic
- `crates/openfang-channels/src/types.rs` (400+ lines) - ChannelAdapter trait, core types
- `crates/openfang-channels/src/discord.rs` (400+ lines) - Discord Gateway adapter
- 50+ individual adapter files

**Channel Adapter Files:**
```
Wave 2: anthropic.rs, claude_code.rs, copilot.rs, fallback.rs, gemini.rs,
         openai.rs, qwen_code.rs, bluesky.rs, dingtalk.rs, dingtalk_stream.rs,
         discord.rs, discourse.rs, email.rs, feishu.rs, flock.rs, formatter.rs,
         gitter.rs, google_chat.rs, gotify.rs, guilded.rs, irc.rs, keybase.rs,
         lib.rs, line.rs, linkedin.rs, mastodon.rs, matrix.rs, mattermost.rs,
         messenger.rs, mumble.rs, nextcloud.rs, nostr.rs, ntfy.rs, pumble.rs,
         reddit.rs, revolt.rs, rocketchat.rs, router.rs, signal.rs, slack.rs,
         teams.rs, telegram.rs, twitch.rs, types.rs, viber.rs, webhook.rs,
         whatsapp.rs, xmpp.rs, zulip.rs
```

### ChannelAdapter Trait (types.rs)

```rust
#[async_trait]
pub trait ChannelAdapter: Send + Sync {
    fn name(&self) -> &str;
    fn channel_type(&self) -> ChannelType;

    // Start the adapter and return a stream of incoming messages
    async fn start(&self) -> Result<Pin<Box<dyn Stream<Item = ChannelMessage> + Send>>, Box<dyn std::error::Error>>;

    // Optional: send to user
    async fn send(&self, user: &ChannelUser, content: ChannelContent) -> Result<(), String>;
    async fn send_in_thread(&self, user: &ChannelUser, content: ChannelContent, thread_id: &str) -> Result<(), String>;

    // Optional: typing indicator
    async fn send_typing(&self, user: &ChannelUser) -> Result<(), String>;

    // Optional: emoji reaction
    async fn send_reaction(&self, user: &ChannelUser, message_id: &str, reaction: &LifecycleReaction) -> Result<(), String>;
}
```

### BridgeManager Dispatch (bridge.rs lines 300-400)

**Concurrency Model:**
```rust
pub async fn start_adapter(&mut self, adapter: Arc<dyn ChannelAdapter>) {
    let stream = adapter.start().await?;
    let semaphore = Arc::new(tokio::sync::Semaphore::new(32));  // Max 32 concurrent

    tokio::spawn(async move {
        let mut stream = std::pin::pin!(stream);
        loop {
            tokio::select! {
                msg = stream.next() => {
                    match msg {
                        Some(message) => {
                            // Spawn concurrent dispatch task
                            tokio::spawn(async move {
                                let _permit = sem.acquire().await?;
                                dispatch_message(&message, &handle, &router, ...).await;
                            });
                        }
                        None => break,
                    }
                }
                shutdown = shutdown.changed() => { if *shutdown.borrow() { break; } }
            }
        }
    });
}
```

**Key insight:** Per-message concurrency (not per-agent) so the stream is never blocked by slow LLM calls. Per-agent serialization is handled by kernel's `agent_msg_locks`.

**Dispatch Message Flow (bridge.rs lines 500+):**
1. Rate limit check
2. RBAC authorization (`handle.authorize_channel_user()`)
3. Agent resolution (by ID or name/routing rules)
4. LLM call via `handle.send_message()`
5. Response formatting
6. Send via adapter
7. Typing indicator loop (4-second refresh)

### Discord Adapter Example (discord.rs)

**WebSocket Gateway Pattern:**
```rust
pub struct DiscordAdapter {
    token: Zeroizing<String>,  // SECURITY: zeroized on drop
    client: reqwest::Client,
    intents: u64,
    bot_user_id: Arc<RwLock<Option<String>>>,
    session_id: Arc<RwLock<Option<String>>>,
    resume_gateway_url: Arc<RwLock<Option<String>>>,
}

async fn start(&self) -> Result<Pin<Box<dyn Stream<Item = ChannelMessage> + Send>>, ...> {
    let gateway_url = self.get_gateway_url().await?;
    let (tx, rx) = mpsc::channel(256);

    tokio::spawn(async move {
        let mut backoff = INITIAL_BACKOFF;
        let mut connect_url = gateway_url;

        loop {
            let ws_stream = tokio_tungstenite::connect_async(&connect_url).await?;
            backoff = INITIAL_BACKOFF;

            // Inner loop: handle gateway events
            'inner: loop {
                let msg = ws_rx.next().await;
                match parse_opcode(msg) {
                    opcode::DISPATCH => { /* handle event */ }
                    opcode::HEARTBEAT => { /* send heartbeat ack */ }
                    opcode::RECONNECT => { break 'inner; }  // Reconnect
                    opcode::INVALID_SESSION => { break 'inner; }
                    opcode::HELLO => { /* start heartbeat interval */ }
                }
            }
            backoff = (backoff * 2).min(MAX_BACKOFF);
        }
    });

    Ok(Box::pin(tokio_stream::channel::ReceiverStream::new(rx)))
}
```

**Exponential Backoff:** Starts at 1s, doubles each failure, caps at 60s.

**Message Sending:** REST API via `api_send_message()` with chunking for >2000 chars.

### Output Formatting

```rust
fn default_output_format_for_channel(channel_type: &str) -> OutputFormat {
    match channel_type {
        "telegram" => OutputFormat::TelegramHtml,
        "slack" => OutputFormat::SlackMrkdwn,
        "wecom" => OutputFormat::PlainText,  // WeCom has no formatting
        _ => OutputFormat::Markdown,
    }
}
```

**Emoji Allowlist (types.rs lines 140-150):**
```rust
pub const ALLOWED_REACTION_EMOJI: &[&str] = &[
    "\u{1F914}",        // 🤔 thinking
    "\u{2699}\u{FE0F}", // ⚙️ tool_use
    "\u{270D}\u{FE0F}", // ✍️ streaming
    "\u{2705}",         // ✅ done
    "\u{274C}",         // ❌ error
    "\u{23F3}",         // ⏳ queued
    "\u{1F504}",        // 🔄 processing
    "\u{1F440}",        // 👀 looking
];
```

### Rate Limiting

```rust
pub struct ChannelRateLimiter {
    buckets: Arc<DashMap<String, Vec<Instant>>>,  // "channel:platform_id" → timestamps
}

impl ChannelRateLimiter {
    pub fn check(&self, channel_type: &str, platform_id: &str, max_per_minute: u32) -> Result<(), String> {
        if max_per_minute == 0 { return Ok(()); }
        let key = format!("{channel_type}:{platform_id}");
        let now = Instant::now();
        let window = Duration::from_secs(60);

        let mut entry = self.buckets.entry(key).or_default();
        entry.retain(|&ts| now.duration_since(ts) < window);  // Evict old

        if entry.len() >= max_per_minute as usize {
            return Err("Rate limit exceeded".to_string());
        }
        entry.push(now);
        Ok(())
    }
}
```

### Security

**1. Bot Token Zeroization (discord.rs line 39):**
```rust
pub struct DiscordAdapter {
    token: Zeroizing<String>,  // Auto-wipes on drop
    ...
}
```

**2. SSRF Protection:** Implemented in `web_fetch` tool, not per-channel.

**3. Per-Channel RBAC:**
```rust
async fn authorize_channel_user(
    &self,
    channel_type: &str,
    platform_id: &str,
    action: &str,
) -> Result<(), String> {
    // Default: allow all (RBAC disabled)
    // Implemented in kernel for actual enforcement
}
```

### Clever Patterns

**1. Concurrent Dispatch with Semaphore (bridge.rs line 322):**
```rust
let semaphore = Arc::new(tokio::sync::Semaphore::new(32));
```
32 concurrent dispatch tasks max. Prevents unbounded memory growth under burst traffic while allowing parallelism.

**2. Typing Indicator Refresh Loop (bridge.rs lines 456-472):**
```rust
fn spawn_typing_loop(adapter: Arc<dyn ChannelAdapter>, sender: ChannelUser) -> JoinHandle<()> {
    tokio::spawn(async move {
        loop {
            tokio::time::sleep(Duration::from_secs(4)).await;
            let _ = adapter.send_typing(&sender).await;
        }
    })
}
```
Refreshes every 4 seconds because Telegram typing indicators expire after ~5 seconds.

**3. Agent Re-resolution on NotFound (bridge.rs lines 487-499):**
```rust
async fn try_reresolution(err: &str, channel_key: &str, handle: &Arc<dyn ChannelBridgeHandle>, router: &Arc<AgentRouter>) -> Option<AgentId> {
    if !err.to_lowercase().contains("agent not found") { return None; }
    let name = router.channel_default_name(channel_key)?;
    handle.find_agent_by_name(&name).await.ok()
}
```
If agent not found, tries re-resolving by the channel's default agent name (stored at bridge startup).

**4. Fire-and-Forget Lifecycle Reactions (bridge.rs lines 437-454):**
```rust
async fn send_lifecycle_reaction(adapter: &dyn ChannelAdapter, user: &ChannelUser, message_id: &str, phase: AgentPhase) {
    let _ = adapter.send_reaction(user, message_id, &reaction).await;
}
```
Non-blocking, errors silently ignored. Reactions are UX polish, not critical path.

### Technical Debt / Concerns

1. **50+ adapter files but each is relatively large** - The discord.rs alone is 400+ lines with WebSocket handling. Adding new adapters requires significant boilerplate.

2. **No adapter-level health monitoring** - BridgeManager tracks tasks but doesn't monitor adapter health. If an adapter silently fails, only visible through missing messages.

3. **`ChannelBridgeHandle` trait has many default no-op implementations** - Most methods return `"Not implemented"` errors. Forces every caller to check for specific error patterns.

4. **WeCom special-casing in formatting (bridge.rs line 410):**
```rust
let formatted = if adapter.name() == "wecom" {
    formatter::format_for_wecom(&text, output_format)
} else {
    formatter::format_for_channel(&text, output_format)
};
```
WeCom has no formatting, gets plain text. This special case leaks into dispatch logic.

5. **Hardcoded 4-second typing refresh interval** - Works for Telegram but may not be optimal for all platforms. Could be configurable per adapter.

---

## Cross-Cutting Observations

### Shared Patterns Across Features

1. **DashMap for concurrent registries** - Used in both `HandRegistry` and `ChannelRateLimiter` for lock-free concurrent access.

2. **`include_str!()` for compile-time embedding** - Used for bundled hands and their SKILL.md files. Enables zero-runtime-I/O for core content.

3. **`Zeroizing<String>` for secrets** - Used for Discord bot tokens and likely other credentials. Automatic memory zeroization on drop.

4. **Exponential backoff with jitter** - Used in Discord WebSocket reconnection, LLM retry logic.

5. **Error classification for smart handling** - `llm_errors::classify_error()` categorizes errors into retryable/billing/format/not_found vs. fatal. Used to drive retry decisions.

6. **Circuit breaker pattern** - Used in LoopGuard (tool call circuit breaker) and ProviderCooldown (LLM request storm prevention).

### Design Philosophy

- **Resilience over performance** - Empty response retry, phantom action detection, interim session saves, fallback chains everywhere.
- **Defense in depth** - Multiple security layers: taint checking, SSRF protection, shell metacharacter blocking, capability gates, approval flows.
- **Zero-cost abstractions where possible** - `include_str!` for compile-time loading, trait-based polymorphism for drivers/adapters.
- **Operational visibility** - Dashboard metrics for hands, typing indicators, lifecycle reactions, delivery receipts.

### Most Complex Code Paths

1. **agent_loop.rs lines 347-896** - The main iteration loop with 5 different stop_reason branches, each with complex error handling.
2. **discord.rs lines 167-370** - WebSocket connection management with exponential backoff, heartbeat, session resume, and event dispatch.
3. **tool_runner.rs lines 99-526** - Single massive match statement handling 50+ tool types with different execution paths.
4. **registry.rs lines 429-636** - Cross-platform binary detection with Python shim avoidance and Chromium hunting in Playwright cache.

### Notable Missing Tests (based on code patterns)

- No tests for actual multi-adapter concurrent dispatch
- No tests for WebSocket resume after INVALID_SESSION
- No tests for the ModelNotFound fallback chain under real network conditions
- No tests for phantom action detection accuracy
