# My Action Items

Based on learnings from the nanobot study, these are actionable items for building a similar project.

## Phase 1: Foundation

### Architecture Decisions

- [ ] **Adopt event-driven architecture with async message bus**
  - Use `asyncio.Queue` for decoupling producers from consumers
  - Implement `InboundMessage` and `OutboundMessage` dataclasses

- [ ] **Use registry/plugin pattern for extensibility**
  - Centralized registries for providers, channels, tools
  - Zero-import discovery via `pkgutil.iter_modules`

- [ ] **Choose Pydantic for configuration**
  - `pydantic-settings` with environment variable support
  - Type validation at boundaries

### Core Dependencies

- [ ] **Python 3.11+ with uv package manager**
  - Fast installation, deterministic builds
  - hatchling for build system

- [ ] **Key libraries to adopt:**
  - `pydantic` + `pydantic-settings` for data validation
  - `loguru` for structured logging
  - `httpx` for HTTP (sync/async)
  - `asyncio` throughout

## Phase 2: Agent Core

### Implement Agent Loop

- [ ] **Main processing loop with:**
  - Concurrent tool execution via `asyncio.gather`
  - Tool context isolation via `contextvars`
  - Max iteration protection (25 default)
  - Streaming with think-block filtering

- [ ] **Session management:**
  - Per-session locks for serial processing
  - Cross-session parallelism
  - JSONL persistence

- [ ] **Memory system:**
  - Two-layer architecture (summary + searchable log)
  - Token-based consolidation
  - Graceful degradation after failures

### Tool System

- [ ] **Abstract Tool base class:**
  ```python
  class Tool(ABC):
      @property @abstractmethod def name(self) -> str
      @property @abstractmethod def description(self) -> str
      @property @abstractmethod def parameters(self) -> dict
      @abstractmethod async def execute(self, **kwargs) -> Any
  ```

- [ ] **Tool registry with:**
  - Parameter validation and casting
  - Error hints for self-correction
  - OpenAI function format export

- [ ] **Security guards for shell execution:**
  - Regex deny patterns for dangerous commands
  - Path traversal prevention
  - SSRF blocking for URLs in commands

## Phase 3: Integrations

### LLM Provider Abstraction

- [ ] **Provider interface:**
  ```python
  class LLMProvider(ABC):
      async def chat(messages, tools, model, max_tokens, temperature) -> LLMResponse
      async def chat_with_retry(...) -> LLMResponse
  ```

- [ ] **Implement OpenAI-compatible provider first** (covers 15+ providers)
- [ ] **Add Anthropic provider** for Claude support
- [ ] **Provider registry with auto-detection** via api_key prefix or api_base URL

### Channel Adapters

- [ ] **Abstract BaseChannel:**
  ```python
  class BaseChannel(ABC):
      async def start(self) -> None
      async def stop(self) -> None
      async def send(self, msg: OutboundMessage) -> None
  ```

- [ ] **Implement 1-2 channels first** (e.g., CLI + Telegram)
- [ ] **Add allowlist authorization** via `is_allowed()` method
- [ ] **Outbound retry with exponential backoff**

### Web Tools

- [ ] **WebSearchTool:**
  - Multi-provider chain with fallback
  - DuckDuckGo as always-available fallback

- [ ] **WebFetchTool:**
  - SSRF protection via blocked network list
  - Redirect limit (5 max)
  - Content sanitization (strip scripts, styles)

## Phase 4: Operations

### Security

- [ ] **SSRF protection module:**
  ```python
  _BLOCKED_NETWORKS = [
      ipaddress.ip_network("127.0.0.0/8"),
      ipaddress.ip_network("10.0.0.0/8"),
      ipaddress.ip_network("169.254.0.0/16"),  # Cloud metadata
      # ... more RFC1918, IPv6
  ]
  ```

- [ ] **Rate limiting** (application-level)
- [ ] **API key encryption at rest** (or use environment variables)
- [ ] **Run as non-root user** in production

### Performance

- [ ] **Concurrency control** via semaphore
- [ ] **Output truncation** for large results
- [ ] **Pagination** for file/directory operations
- [ ] **Session caching** in memory

### Testing

- [ ] **pytest with pytest-asyncio**
- [ ] **Fake objects** instead of mocks where appropriate
- [ ] **Security test coverage** for SSRF, path traversal, pattern blocking
- [ ] **Integration tests** for message flows

### CI/CD

- [ ] **GitHub Actions** with multiple Python versions
- [ ] **Pre-commit hooks** with ruff formatting
- [ ] **pip-audit** for dependency security

## Anti-Patterns to Avoid

- [ ] **Do NOT use global mutable state** for configuration
- [ ] **Do NOT store secrets in plain text** - use env vars or keyring
- [ ] **Do NOT skip rate limiting** - implement at application level
- [ ] **Do NOT use regex alone** for shell security - consider true sandboxing
- [ ] **Do NOT forget memory deletion** - implement redact capability

## Nice-to-Have Features

- [ ] Skills system with markdown prompts
- [ ] Cron scheduling with timezone support
- [ ] Heartbeat for periodic tasks
- [ ] MCP protocol support
- [ ] Multi-instance isolation

## Suggested Technology Choices

| Aspect | Recommendation |
|--------|----------------|
| Language | Python 3.11+ |
| Package Manager | uv |
| CLI | Typer |
| Data Validation | Pydantic v2 |
| Logging | Loguru |
| HTTP | httpx |
| Testing | pytest + pytest-asyncio |
| Linting | Ruff |
| Build | hatchling |
