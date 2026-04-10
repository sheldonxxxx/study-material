# Feature Deep Dive - Batch 4 (Features 10-12)

## Feature 10: CLI and SDK Tools

### Core Implementation

**Primary Files:**
- `swarms/cli/main.py` (57KB, ~1676 lines) - Main CLI entry point
- `swarms/cli/utils.py` (20KB) - CLI utilities and helpers
- `swarms/structs/agent_loader.py` (210 lines) - Unified agent loader
- `swarms/utils/` (29 utility modules) - SDK helper functions

### CLI Architecture

The CLI uses `argparse` with a custom help action for rich terminal output. The main entry point `main()` (line 1616) displays ASCII art, parses arguments, and routes commands.

**Command Structure:**
```python
command_choices = [
    "onboarding",        # Environment setup check
    "get-api-key",       # Open browser for API keys
    "check-login",       # Verify authentication
    "run-agents",        # Execute from YAML config
    "load-markdown",     # Load from markdown files
    "agent",             # Create/run custom agent
    "chat",              # Interactive chat mode
    "upgrade",           # Update swarms package
    "autoswarm",         # Auto-generate swarm config
    "setup-check",       # Comprehensive env check
    "llm-council",       # Multi-agent collaboration
    "heavy-swarm",       # Complex task analysis
]
```

**Handler Pattern:**
Each command has a dedicated handler function:
- `handle_agent()` (line 1278) - Agent creation with validation
- `handle_chat()` (line 1500) - Interactive chat via `auto_chat_agent()`
- `handle_autoswarm()` (line 1396) - Swarm auto-generation
- `handle_heavy_swarm()` (line 267) - HeavySwarm orchestration
- `handle_llm_council()` (line 380) - LLM Council execution
- `handle_load_markdown()` (line 1239) - Markdown agent loading

### CLI Utilities (`swarms/cli/utils.py`)

```python
# Color scheme (all red-based)
COLORS = {
    "primary": "red",
    "secondary": "red",
    "accent": "white",
    # ...
}

# API key detection
def _detect_active_provider() -> str:
    # Detects OpenAI, Anthropic, Groq, Google, Cohere, Mistral, Together AI
```

Key utilities:
- `show_ascii_art()` - Claude Code-style CLI header with version
- `SwarmCLIError` - Custom exception class
- `show_error()` - Formatted error display with troubleshooting tips
- `run_setup_check()` - Environment validation
- `check_login()` - Authentication verification

### AgentLoader (`swarms/structs/agent_loader.py`)

Unified loader supporting multiple file formats:

```python
class AgentLoader:
    def load_agents_from_markdown(...)   # Markdown with YAML frontmatter
    def load_agents_from_yaml(...)       # YAML configuration
    def load_agents_from_csv(...)       # CSV-based agents
    def auto(...)                        # Auto-detect by extension
    def load_multiple_agents(...)        # Batch loading
```

Delegation pattern: `AgentLoader` wraps specialized loaders:
- `MarkdownAgentLoader` from `swarms.utils.agent_loader_markdown`
- `create_agents_from_yaml` from `swarms.agents.create_agents_from_yaml`
- `CSVAgentLoader` from `swarms.structs.csv_to_agent`

### SDK Utilities (`swarms/utils/`)

Key utilities relevant to CLI/SDK:

| File | Size | Purpose |
|------|------|---------|
| `litellm_wrapper.py` | 59KB | Unified LLM interface (OpenAI, Anthropic, Gemini, etc.) |
| `formatter.py` | 31KB | Output formatting |
| `agent_loader_markdown.py` | 18KB | Markdown-based agent loading |
| `fetch_prompts_marketplace.py` | 6KB | Swarms marketplace integration |
| `swarm_autosave.py` | 12KB | Agent state persistence |
| `generate_keys.py` | 1KB | API key generation |

### Notable Patterns

**Lazy Imports:** Used extensively to avoid circular dependencies:
```python
# Example from agent_loader.py
def load_agent_from_markdown(self, file_path: str, **kwargs) -> "Agent":
    from swarms.structs.agent import Agent  # Lazy import
    config = self.parse_markdown_file(file_path)
    agent = Agent(**agent_fields)
```

**Rich Progress Indicators:** CLI uses `rich.Progress` with spinners for async operations.

**Error Handling:** Contextual error messages with troubleshooting steps (e.g., context_length_exceeded, api_key issues).

### Clever Solutions

1. **Auto-format detection:** `AgentLoader.auto()` routes based on file extension
2. **Concurrent markdown loading:** Uses `ThreadPoolExecutor` with configurable workers
3. **Marketplace integration:** Can fetch system prompts via `marketplace-prompt-id` flag
4. **Multi-model defaults:** `gpt-5.4` for chat, `gpt-4` for agent creation

### Technical Debt / Concerns

1. **File location:** `agent_loader_markdown.py` is in `swarms/agents/` but logically belongs in `swarms/utils/`
2. **Inconsistent naming:** `create_agents_from_yaml` imported from `swarms.agents` despite being a utility
3. **mcp_url type:** `MarkdownAgentConfig.mcp_url` typed as `Optional[int]` but URLs are strings (line 31)
4. **No validation on file paths:** Limited sanitization of user-provided paths

---

## Feature 11: X402 Payment Protocol

### Implementation Status: Documentation/Examples Only

The X402 payment protocol is **not implemented in the core swarms package**. It is purely an external integration demonstrated through examples and documentation.

### Files

| File | Purpose |
|------|---------|
| `examples/guides/x402_examples/README.md` | Usage documentation |
| `examples/guides/x402_examples/research_agent_x402_example.py` | Working example |
| `examples/guides/x402_examples/agent_integration/x402_agent_buying.py` | Agent purchasing example |
| `examples/guides/x402_examples/agent_integration/x402_discovery_query.py` | Discovery query example |
| `examples/guides/x402_examples/memecoin_agent_x402.py` | Memecoin agent example |
| `docs/examples/x402_payment_integration.md` | Integration guide |

### How X402 Integration Works

**Protocol Overview:** X402 is Coinbase's payment protocol for monetizing APIs with crypto. It adds payment middleware that returns HTTP 402 "Payment Required" until the client provides payment proof.

**Swarms Integration Pattern:**

```python
from x402.fastapi.middleware import require_payment
from swarms import Agent
from swarms_tools import exa_search

# 1. Create agent with tools
research_agent = Agent(
    agent_name="Research-Agent",
    system_prompt="You are an expert research analyst...",
    model_name="gpt-5.4",
    max_loops=1,
    tools=[exa_search],
)

# 2. Apply payment middleware
app.middleware("http")(
    require_payment(
        path="/research",
        price="$0.01",
        pay_to_address="0xYourWalletAddressHere",
        network_id="base-sepolia",
        description="AI-powered research agent",
        input_schema={...},
        output_schema={...},
    )
)

# 3. Define endpoint
@app.get("/research")
async def conduct_research(query: str):
    result = research_agent.run(query)
    return {"research": result}
```

### Payment Flow

```
Client Request → 402 Payment Required + payment instructions
       ↓
Client Payment (via x402 SDK)
       ↓
Client Retry with payment proof
       ↓
Server verifies payment
       ↓
Server returns research results
```

### Network/Configuration

| Environment | Network | Notes |
|-------------|---------|-------|
| Testnet | `base-sepolia` | Free facilitator, test USDC only |
| Production | `base` | Requires CDP API credentials |

**Required Setup:**
```bash
# Dependencies
pip install x402 fastapi uvicorn python-dotenv swarms-tools

# Environment
OPENAI_API_KEY=...
EXA_API_KEY=...
WALLET_ADDRESS=0x...  # EVM-compatible wallet
```

### Key Dependencies (External)

```python
x402>=0.1.0       # Payment protocol
fastapi>=0.100    # Web framework
uvicorn>=0.23     # ASGI server
swarms_tools      # Tools library (exa_search, etc.)
```

### Observations

1. **No core implementation:** X402 is purely example/documentation - no `swarms.x402` module
2. **Minimal swarms coupling:** Agents are used as-is; x402 is orthogonal
3. **Pattern is generalizable:** Any FastAPI endpoint can wrap any swarms component
4. **Documentation is thorough:** Examples cover testnet, production, and troubleshooting

---

## Feature 12: Agent Skills (Anthropic-compatible)

### Core Implementation

**Primary Files:**
- `swarms/agents/agent_loader_markdown.py` (498 lines) - Markdown loader
- `swarms/structs/agent_loader.py` (210 lines) - AgentLoader wrapper
- `examples/single_agent/agent_skill_examples/` - Example skills

### Format Specification

Agent skills use **markdown files with YAML frontmatter** following the Claude Code sub-agent format:

```markdown
---
name: skill-name
description: Skill description
model_name: claude-sonnet-4-20250514
temperature: 0.7
max_loops: 1
---

# Skill Name

System prompt content goes here as markdown...

Can include:
- Structured guidelines
- Step-by-step processes
- Example outputs
- Best practices
```

### MarkdownAgentConfig Schema

```python
class MarkdownAgentConfig(BaseModel):
    name: Optional[str] = None
    description: Optional[str] = None
    model_name: Optional[str] = DEFAULT_MODEL  # "gpt-4.1"
    temperature: Optional[float] = Field(default=0.1, ge=0.0, le=2.0)
    mcp_url: Optional[int] = None  # BUG: should be Optional[str]
    system_prompt: Optional[str] = None
    max_loops: Optional[int] = Field(default=1, ge=1)
    autosave: Optional[bool] = False
    dashboard: Optional[bool] = False
    verbose: Optional[bool] = False
    dynamic_temperature_enabled: Optional[bool] = False
    saved_state_path: Optional[str] = None
    user_name: Optional[str] = "default_user"
    retry_attempts: Optional[int] = Field(default=3, ge=1)
    context_length: Optional[int] = Field(default=100000, ge=1000)
    return_step_meta: Optional[bool] = False
    output_type: Optional[str] = "str"
    auto_generate_prompt: Optional[bool] = False
    streaming_on: Optional[bool] = False
```

### MarkdownAgentLoader

```python
class MarkdownAgentLoader:
    def parse_yaml_frontmatter(self, content: str) -> Dict[str, Any]:
        # Extracts --- YAML block --- from markdown

    def parse_markdown_file(self, file_path: str) -> MarkdownAgentConfig:
        # Returns config from YAML frontmatter

    def load_agent_from_markdown(self, file_path: str, **kwargs) -> Agent:
        # Creates Agent from markdown file

    def load_agents_from_markdown(self, file_paths, concurrent=True, ...) -> List[Agent]:
        # Batch loading with ThreadPoolExecutor
```

### Example Skills

**Code Review Skill** (`agent_skill_examples/code-review/SKILL.md`):
- Comprehensive review checklist (quality, security, performance)
- OWASP Top 10 vulnerability checks
- DRY/SOLID principles verification
- Review format template

**Financial Analysis Skill** (`agent_skill_examples/financial-analysis/SKILL.md`):
- DCF modeling methodology
- Financial ratio analysis framework
- Valuation models (comparable company, precedent transactions)
- Sensitivity analysis guidelines

**Finance Advisor** (`agent_skill_examples/finance_advisor.md`):
- Investment strategies and portfolio management
- Risk assessment and diversification
- Tax optimization and retirement planning

### Loading Flow

```python
# Via CLI
swarms load-markdown --markdown-path ./agents/

# Via AgentLoader
loader = AgentLoader()
agents = loader.load_agents_from_markdown("./path/to/skills/")

# Via convenience function
from swarms.utils.agent_loader_markdown import load_agent_from_markdown
agent = load_agent_from_markdown("code-review.md", max_loops=2)
```

### Field Mapping

```python
field_mapping = {
    "name": "agent_name",      # Frontmatter 'name' → Agent 'agent_name'
    "description": None,       # Not used by Agent
    "mcp_url": "mcp_url",      # Passed through (mis-typed as int)
}
```

### Notable Features

1. **Concurrent loading:** Uses `os.cpu_count() * 2` workers for batch processing
2. **File size limits:** Default 10MB max per file
3. **Timeout handling:** 5 minute overall, 1 minute per agent
4. **Graceful degradation:** Skips failed files with warnings, continues loading

### Relationship to Feature 10 (CLI)

Agent skills are the **content format** that the CLI's `load-markdown` command consumes:

```python
# CLI handler (line 1239)
def handle_load_markdown(args):
    agents = load_markdown_agents(args.markdown_path, concurrent=args.concurrent)

# Calls AgentLoader.load_multiple_agents()
# Which calls MarkdownAgentLoader.load_agents_from_markdown()
```

### Bug: mcp_url Type Error

```python
# In agent_loader_markdown.py, line 31:
mcp_url: Optional[int] = None  # WRONG - should be Optional[str]

# But URL is always a string, causing potential issues
# when Agent constructor receives a string for this field
```

### Technical Debt

1. **File location:** `agent_loader_markdown.py` in `swarms/agents/` instead of `swarms/utils/`
2. **Type bug:** `mcp_url` typed as `int` instead of `str`
3. **No validation:** System prompt validated but `mcp_url` has no validation despite being passed through
4. **Limited error recovery:** Only logs warnings, no retry mechanism for transient failures

### Anthropic Compatibility

The format is explicitly designed to be compatible with Claude Code's sub-agent YAML frontmatter:
- Same field names (`name`, `description`, `model_name`, `temperature`)
- Same markdown content structure
- Pattern allows importing Claude Code skills directly into swarms
