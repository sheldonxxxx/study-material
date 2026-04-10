# Design Patterns in Moltis

## Pattern Index

| Pattern | Location | Purpose |
|---------|----------|---------|
| Plugin Architecture | `crates/channels` | Runtime platform integration |
| Service Trait Bundle | `crates/service-traits` | Dependency injection |
| Registry Pattern | `crates/gateway` | Centralized component lookup |
| Builder Pattern | Throughout | Object construction |
| Noop/Stub Pattern | `crates/service-traits` | Graceful degradation |
| Feature Gating | `Cargo.toml` | Compile-time conditional |
| Static/Dynamic Dispatch | Throughout | Performance optimization |
| Provider Chain | `crates/agents` | Failover and fallback |
| Approval Workflow | `crates/tools` | Multi-layer security |
| Event Sourcing | `crates/sessions` | Audit trail |

---

## 1. Plugin Architecture

**Problem**: Need to support multiple messaging platforms (Discord, Slack, Telegram, WhatsApp, MS Teams) with platform-specific implementations.

**Solution**: Define a `ChannelPlugin` trait that each platform implements. Gateway loads plugins at runtime without compile-time knowledge.

**Evidence** - `crates/channels/src/lib.rs`:

```rust
/// Main channel plugin trait
#[async_trait]
pub trait ChannelPlugin: Send + Sync {
    /// Unique channel type identifier
    fn channel_type(&self) -> ChannelType;

    /// Start handling an account
    async fn start_account(&self, account_id: AccountId, config: ChannelConfig) -> Result<()>;

    /// Stop handling an account
    async fn stop_account(&self, account_id: AccountId) -> Result<()>;

    /// Get outbound message sender
    fn outbound(&self) -> Arc<dyn ChannelOutbound>;

    /// Get channel status interface
    fn status(&self) -> Arc<dyn ChannelStatus>;

    /// Event sink for inbound events
    fn event_sink(&self) -> ChannelEventSink;
}
```

**Implementation** - `crates/discord/src/lib.rs`:

```rust
pub struct DiscordChannel {
    client: Arc<DiscordClient>,
    event_sink: ChannelEventSink,
}

#[async_trait]
impl ChannelPlugin for DiscordChannel {
    fn channel_type(&self) -> ChannelType {
        ChannelType::Discord
    }

    async fn start_account(&self, account_id: AccountId, config: ChannelConfig) -> Result<()> {
        // Initialize Discord client and start event loop
    }
    // ...
}
```

**Usage** - `crates/gateway/src/channel.rs`:

```rust
pub struct ChannelRegistry {
    plugins: HashMap<ChannelType, Arc<dyn ChannelPlugin>>,
    running_accounts: HashMap<AccountId, RunningAccount>,
}

impl ChannelRegistry {
    pub async fn start_account(&self, channel_type: ChannelType, config: ChannelConfig) -> Result<AccountId> {
        let plugin = self.plugins.get(&channel_type)
            .ok_or_else(|| anyhow!("No plugin for channel type: {:?}", channel_type))?;

        plugin.start_account(account_id.clone(), config).await
    }
}
```

---

## 2. Service Trait Bundle

**Problem**: Gateway needs access to many domain services (agents, sessions, channels, config, chat, skills, MCP, browser) without tight coupling.

**Solution**: Define each service as a trait with a `Noop*` default implementation. Bundle all trait objects in a single `GatewayServices` struct for dependency injection.

**Evidence** - `crates/service-traits/src/lib.rs`:

```rust
/// All domain services bundled together
#[derive(Clone)]
pub struct Services {
    pub agent: Arc<dyn AgentService>,
    pub session: Arc<dyn SessionService>,
    pub channel: Arc<dyn ChannelService>,
    pub config: Arc<dyn ConfigService>,
    pub chat: Arc<dyn ChatService>,
    pub skills: Arc<dyn SkillsService>,
    pub mcp: Arc<dyn McpService>,
    pub browser: Arc<dyn BrowserService>,
    pub vault: Arc<dyn VaultService>,
    pub memory: Arc<dyn MemoryService>,
    pub cron: Arc<dyn CronService>,
    pub oauth: Arc<dyn OAuthService>,
    pub node: Arc<dyn NodeService>,
    pub presence: Arc<dyn PresenceService>,
    pub billing: Arc<dyn BillingService>,
    pub email: Arc<dyn EmailService>,
    pub sms: Arc<dyn SmsService>,
}

// Trait with Noop default for graceful degradation
pub trait AgentService: Send + Sync {
    fn run(&self, params: AgentRunParams) -> InstrumentedFuture<AgentRunResult>;

    // Override for actual implementation
    fn resolve_skill(&self, skill_id: &str) -> BoxFuture<Option<ResolvedSkill>> {
        async { None }.boxed()
    }
}

pub struct NoopAgentService;

impl AgentService for NoopAgentService {}
```

---

## 3. Registry Pattern

**Problem**: Need centralized lookup and management for channels, skills, tools, and nodes.

**Solution**: Each registry provides `register`, `get`, `list`, and `remove` operations with thread-safe interior mutability.

**Evidence** - `crates/gateway/src/channel.rs`:

```rust
pub struct ChannelRegistry {
    plugins: HashMap<ChannelType, Arc<dyn ChannelPlugin>>,
    running_accounts: HashMap<AccountId, RunningAccount>,
    status_callbacks: Vec<ChannelStatusCallback>,
}

impl ChannelRegistry {
    pub fn register(&mut self, plugin: Arc<dyn ChannelPlugin>) {
        let channel_type = plugin.channel_type();
        self.plugins.insert(channel_type, plugin);
    }

    pub fn get(&self, channel_type: &ChannelType) -> Option<Arc<dyn ChannelPlugin>> {
        self.plugins.get(channel_type).cloned()
    }

    pub fn list(&self) -> Vec<ChannelType> {
        self.plugins.keys().cloned().collect()
    }
}
```

**Evidence** - `crates/skills/src/registry.rs`:

```rust
pub trait SkillRegistry: Send + Sync {
    fn discover(&self, source: SkillSource) -> BoxFuture<()>;
    fn list(&self) -> BoxFuture<Vec<DiscoveredSkill>>;
    fn resolve(&self, skill_id: &str) -> BoxFuture<Option<ResolvedSkill>>;
    fn reload(&self, skill_id: &str) -> BoxFuture<Result<()>>;
}

pub struct SkillRegistryImpl {
    skills: RwLock<HashMap<SkillId, DiscoveredSkill>>,
    sources: Vec<Box<dyn SkillSource>>,
}

impl SkillRegistry for SkillRegistryImpl {
    fn list(&self) -> BoxFuture<Vec<DiscoveredSkill>> {
        let skills = self.skills.read().unwrap().values().cloned().collect();
        async move { skills }.boxed()
    }
}
```

---

## 4. Builder Pattern

**Problem**: Complex object construction with many optional parameters.

**Solution**: Use builder methods with `Default` and method chaining for object construction.

**Evidence** - `crates/agents/src/model.rs`:

```rust
/// Builder for ChatMessage
impl ChatMessage {
    pub fn system(content: impl Into<String>) -> Self {
        ChatMessage::System { content: content.into() }
    }

    pub fn user(content: impl Into<UserContent>) -> Self {
        ChatMessage::User { content: content.into() }
    }

    pub fn assistant(content: impl Into<String>) -> Self {
        ChatMessage::Assistant {
            content: content.into(),
            tool_calls: None,
        }
    }

    pub fn tool(tool_call_id: impl Into<String>, content: impl Into<String>) -> Self {
        ChatMessage::Tool {
            tool_call_id: tool_call_id.into(),
            content: content.into(),
        }
    }
}

impl ChatMessage {
    pub fn with_tool_calls(mut self, tool_calls: Vec<ToolCall>) -> Self {
        if let ChatMessage::Assistant { tool_calls: tc, .. } = &mut self {
            *tc = Some(tool_calls);
        }
        self
    }
}
```

**Evidence** - `crates/tools/src/exec.rs`:

```rust
#[derive(Debug, Clone, Default)]
pub struct ExecOpts {
    pub timeout_ms: Option<u64>,
    pub environment: HashMap<String, String>,
    pub working_directory: Option<PathBuf>,
    pub user: Option<String>,
    pub group: Option<String>,
    pub max_output_bytes: Option<usize>,
}

impl ExecOpts {
    pub fn timeout_ms(mut self, ms: u64) -> Self {
        self.timeout_ms = Some(ms);
        self
    }

    pub fn env(mut self, key: impl Into<String>, value: impl Into<String>) -> Self {
        self.environment.insert(key.into(), value.into());
        self
    }

    pub fn cwd(mut self, path: impl Into<PathBuf>) -> Self {
        self.working_directory = Some(path.into());
        self
    }
}
```

---

## 5. Noop/Stub Pattern

**Problem**: System should start and function even when optional services are not configured.

**Solution**: Every service trait has a `Noop*` implementation that returns empty/default responses.

**Evidence** - `crates/service-traits/src/lib.rs`:

```rust
// Noop implementations for all services
pub struct NoopAgentService;
impl AgentService for NoopAgentService {}

pub struct NoopSessionService;
impl SessionService for NoopSessionService {
    fn list_sessions(&self, _params: ListSessionsParams) -> BoxFuture<Vec<Session>> {
        async { Vec::new() }.boxed()
    }
}

pub struct NoopSkillsService;
impl SkillsService for NoopSkillsService {
    fn list_skills(&self) -> BoxFuture<Vec<Skill>> {
        async { Vec::new() }.boxed()
    }
}

// Factory for creating Noop bundle
impl Services {
    pub fn noop() -> Self {
        Self {
            agent: Arc::new(NoopAgentService),
            session: Arc::new(NoopSessionService),
            channel: Arc::new(NoopChannelService),
            skills: Arc::new(NoopSkillsService),
            // ... all Noop implementations
        }
    }
}
```

**Usage** - Gateway initializes with Noop services, replaces with real implementations as they become available.

---

## 6. Feature Gating

**Problem**: Many LLM providers available, but users only need a subset. Avoid compile-time bloat.

**Solution**: Feature flags in `Cargo.toml` control which provider implementations are compiled.

**Evidence** - `crates/providers/Cargo.toml`:

```toml
[features]
default = ["provider-openai", "provider-anthropic"]
provider-openai = ["async-openai", "dep:tokio"]
provider-anthropic = ["dep:reqwest"]
provider-github-copilot = ["dep:github-copilot"]
provider-kimi = ["dep:kimi-sdk"]
provider-gemini = ["dep:genai"]
provider-gguf = ["dep:llama"]
```

**Usage**:

```rust
// Only compiled if provider-openai feature enabled
#[cfg(feature = "provider-openai")]
mod openai;

#[cfg(feature = "provider-openai")]
pub use openai::OpenAiProvider;
```

**Activation** - `crates/providers/src/lib.rs`:

```rust
pub struct LlmProviderRegistry {
    providers: HashMap<String, Arc<dyn LlmProvider>>,
}

impl LlmProviderRegistry {
    #[cfg(feature = "provider-openai")]
    pub fn with_openai(self) -> Self {
        self.register(OpenAiProvider::new())
    }

    #[cfg(feature = "provider-anthropic")]
    pub fn with_anthropic(self) -> Self {
        self.register(AnthropicProvider::new())
    }
}
```

---

## 7. Static/Dynamic Dispatch Optimization

**Problem**: Need both performance (hot paths) and flexibility (plugin systems).

**Solution**: Static dispatch (`Arc<T>`) for known types, dynamic dispatch (`dyn Trait`) for plugins and extensions.

**Evidence** - `crates/gateway/src/state.rs`:

```rust
// Static dispatch - known at compile time
pub struct AppState {
    pub sessions: Arc<SessionStore>,        // SessionStore is concrete type
    pub auth: Arc<AuthResolver>,            // AuthResolver is concrete type
}

// Dynamic dispatch - runtime polymorphism
pub struct Services {
    pub agent: Arc<dyn AgentService>,       // Plugin implementations vary
    pub channel: Arc<dyn ChannelService>,   // Different channels loaded
    pub skills: Arc<dyn SkillsService>,      // Skills discovered at runtime
}
```

**Evidence** - `crates/agents/src/runner.rs`:

```rust
// Static dispatch in hot path
async fn run_impl<S: SessionService>(
    &self,
    session_service: &S,
    provider: &dyn LlmProvider,  // Dynamic for provider flexibility
    messages: &[ChatMessage],
) -> Result<AgentRunResult> {
    // Use concrete session service for performance
    let session = session_service.get_session(&self.session_key).await?;
    // Use trait object for provider abstraction
    let response = provider.complete(messages, tools).await?;
}
```

---

## 8. Provider Chain (Failover Pattern)

**Problem**: LLM providers can fail or be rate-limited. Need automatic failover.

**Solution**: `ProviderChain` tries providers in sequence until one succeeds.

**Evidence** - `crates/agents/src/provider_chain.rs`:

```rust
pub struct ProviderChain {
    providers: Vec<ProviderWithConfig>,
    retry_config: RetryConfig,
}

pub struct ProviderWithConfig {
    pub provider: Arc<dyn LlmProvider>,
    pub config: ProviderConfig,
    pub priority: usize,
}

impl ProviderChain {
    pub async fn complete(
        &self,
        request: &CompletionRequest,
    ) -> Result<CompletionResult> {
        let mut last_error = None;

        for provider_config in &self.providers {
            match provider_config.provider.complete(request).await {
                Ok(result) => return Ok(result),
                Err(e) => {
                    last_error = Some(e);
                    continue;  // Try next provider
                }
            }
        }

        Err(anyhow!("All providers failed: {:?}", last_error))
    }
}
```

---

## 9. Approval Workflow (Policy Chain)

**Problem**: Tool execution must be gated by user approval with multiple policy layers.

**Solution**: `ApprovalManager` checks policies in order until one blocks or all pass.

**Evidence** - `crates/tools/src/approval.rs`:

```rust
pub struct ApprovalManager {
    policies: Vec<Box<dyn ApprovalPolicy>>,
    sandbox_factory: Arc<dyn SandboxFactory>,
}

#[async_trait]
pub trait ApprovalPolicy: Send + Sync {
    async fn check(&self, request: &ApprovalRequest) -> ApprovalResult;
}

pub struct MultiLayerPolicy {
    // Policy chain evaluated in order
    layers: Vec<Box<dyn ApprovalPolicy>>,
}

#[async_trait]
impl ApprovalPolicy for MultiLayerPolicy {
    async fn check(&self, request: &ApprovalRequest) -> ApprovalResult {
        for layer in &self.layers {
            match layer.check(request).await {
                ApprovalResult::Denied(reason) => return ApprovalResult::Denied(reason),
                ApprovalResult::Pending => return ApprovalResult::Pending,  // Stop and ask user
                ApprovalResult::Approved => continue,  // Check next layer
            }
        }
        ApprovalResult::Approved
    }
}
```

**Policy layers:**
1. Global allow/deny (config)
2. Per-agent policy
3. Per-provider policy
4. Per-sender policy
5. Sandbox enforcement

---

## 10. Event Sourcing Ready

**Problem**: Need audit trail for messages and session history.

**Solution**: All messages stored in `message_log` table with timestamps and metadata.

**Evidence** - `crates/sessions/src/message_log.rs`:

```rust
pub struct MessageLog {
    pool: sqlx::SqlitePool,
}

pub struct Message {
    pub id: MessageId,
    pub session_key: SessionKey,
    pub role: MessageRole,
    pub content: MessageContent,
    pub created_at: DateTime<Utc>,
    pub metadata: Option<Value>,
}

impl MessageLog {
    pub async fn append(&self, message: &Message) -> Result<MessageId> {
        sqlx::query_as!(
            MessageId,
            r#"INSERT INTO message_log (session_key, role, content, metadata)
               VALUES ($1, $2, $3, $4)"#,
            message.session_key.as_str(),
            message.role,
            message.content,
            message.metadata,
        )
        .fetch_one(&self.pool)
        .await
    }

    pub async fn list(&self, session_key: &SessionKey, limit: usize) -> Result<Vec<Message>> {
        sqlx::query_as!(
            Message,
            r#"SELECT * FROM message_log
               WHERE session_key = $1
               ORDER BY created_at ASC
               LIMIT $2"#,
            session_key.as_str(),
            limit as i64,
        )
        .fetch_all(&self.pool)
        .await
    }
}
```

**Events emitted:**
```rust
enum SessionEvent {
    MessageAdded { session_key, message_id },
    RunStarted { session_key, run_id },
    RunCompleted { session_key, run_id, usage },
    PresenceChanged { session_key, presence },
}
```

---

## Pattern Usage Summary

| Pattern | Crates Using It | Purpose |
|---------|----------------|---------|
| Plugin Architecture | `channels`, `discord`, `slack`, `telegram`, `whatsapp`, `msteams` | Platform abstraction |
| Service Trait Bundle | `gateway`, `service-traits` | Dependency injection |
| Registry Pattern | `gateway`, `skills`, `tools`, `agents` | Component management |
| Builder Pattern | `agents`, `tools`, `config` | Object construction |
| Noop/Stub Pattern | `service-traits` | Graceful degradation |
| Feature Gating | `providers` | Compile-time selection |
| Static/Dynamic Dispatch | Throughout | Performance + flexibility |
| Provider Chain | `agents` | Failover |
| Approval Workflow | `tools` | Security |
| Event Sourcing | `sessions` | Audit trail |
