# My Action Items: Building a Multi-Agent Framework

**Based on:** Lessons learned from swarms project analysis
**Date:** 2026-03-26

---

## Priority Framework

| Priority | Definition | Timeline |
|----------|-----------|----------|
| **P0** | Critical - will cause security/correctness issues | Immediate |
| **P1** | High - significant improvement to correctness/maintainability | Before v0.1 |
| **P2** | Medium - good engineering practice | Before v1.0 |
| **P3** | Nice to have - optimization or convenience | Post-v1.0 |

---

## P0: Security & Correctness (Immediate)

### P0.1: Fix SSL Verification Default
**Source:** SSL verification disabled by default in LiteLLM wrapper
**Action:**
```python
# BAD
ssl_verify: bool = False

# GOOD
ssl_verify: bool = True  # Secure default
# If user needs to disable:
ssl_verify: bool = os.getenv("SSL_VERIFY", "true").lower() != "false"
```
**Verification:** Code review, security audit

---

### P0.2: Remove Side Effects from Import
**Source:** Telemetry bootup on `swarms/__init__.py` import
**Action:**
```python
# __init__.py should only do:
from swarms.env import load_swarms_env
load_swarms_env()  # Still in init - load env but NOT telemetry

# Telemetry should be opt-in only:
# from swarms.telemetry import enable_telemetry  # User calls this
```
**Verification:** `python -c "import mypackage; print('no side effects')"`

---

### P0.3: Add Input Size Limits
**Source:** No input size limits on agent prompts
**Action:**
```python
class Agent:
    MAX_PROMPT_LENGTH = 100_000  # 100k chars
    MAX_TOOLS = 50

    def run(self, task: str, ...):
        if len(task) > self.MAX_PROMPT_LENGTH:
            raise ValueError(f"Prompt exceeds {self.MAX_PROMPT_LENGTH} chars")
```

**Verification:** Fuzzing tests with oversized inputs

---

## P1: Architecture & Design (Before v0.1)

### P1.1: Break Monolithic Agent Class
**Source:** 252KB agent.py with 1900+ lines
**Action:** Decompose into composed classes:

```
Agent (composition root)
├── LLMClient (litellm wrapper)
├── ToolRegistry (tool management)
├── MemoryManager (conversation + RAG)
├── HandoffRouter (agent delegation)
└── ExecutionLoop (autonomous loop logic)
```

**Target max file size:** 500 lines per class
**Verification:** `wc -l` per file, enforce in CI

---

### P1.2: Enforce Type Safety
**Source:** Type annotations present but mypy not configured
**Action:**
```toml
# pyproject.toml
[tool.mypy]
python_version = "3.11"
strict = true
disallow_untyped_defs = true
warn_return_any = true
warn_unused_ignores = true

[tool.poetry.group.typecheck.dependencies]
mypy = ">=1.0"
```

**Verification:** `mypy src/` passes in CI

---

### P1.3: Fix Configuration Bugs
**Source:** Black target-version py38 but project requires py310+
**Action:**
```toml
# BAD
target-version = ["py38"]

# GOOD
target-version = ["py311"]  # Match project constraint
line-length = 100  # Reasonable, not restrictive 70
```

**Verification:** `black --check` in CI

---

### P1.4: Standardize Error Handling
**Source:** Silent failures with broad `except Exception`
**Action:**
```python
# Distinguish error types
class AgentError(Exception): ...
class AgentLLMError(AgentError): ...
class AgentToolError(AgentError): ...
class AgentTimeoutError(AgentError): ...

# When to catch vs propagate:
# - Network errors: retry with backoff, then propagate
# - Validation errors: propagate immediately (can't recover)
# - Tool errors: log, try fallback tool, propagate if no fallback
```

**Verification:** Code review checklist, error handling tests

---

### P1.5: Implement LiteLLM-Style Abstraction
**Source:** LiteLLM as brilliant vendor-agnostic pattern
**Action:**
```python
class LLMClient:
    def __init__(self, model_name: str, api_key: str = None):
        self.model_name = model_name
        self.client = self._create_client(model_name)

    def _create_client(self, model_name: str):
        # Route to appropriate provider
        if "gpt" in model_name:
            return OpenAIClient(api_key)
        elif "claude" in model_name:
            return AnthropicClient(api_key)
        # ... etc
        # OR use litellm directly

    def complete(self, prompt: str, **kwargs) -> str:
        return self.client.complete(prompt, **kwargs)
```

**Verification:** Test with multiple providers (mock or test keys)

---

## P2: Engineering Excellence (Before v1.0)

### P2.1: Proper pytest Setup
**Source:** Custom test runners bypassing pytest
**Action:**
```python
# tests/conftest.py
import pytest
from unittest.mock import MagicMock

@pytest.fixture
def mock_llm():
    return MagicMock(complete=lambda p: "mock response")

@pytest.fixture
def agent_with_mock_llm(mock_llm):
    return Agent(llm=mock_llm)

# tests/test_agent.py
def test_agent_run(agent_with_mock_llm):
    result = agent_with_mock_llm.run("test task")
    assert result == "mock response"
```

**Verification:** `pytest tests/ -v` with coverage report

---

### P2.2: Add conftest.py Shared Fixtures
**Source:** No shared fixtures observed
**Action:** Create fixtures for:
- Mock LLM clients (success, failure, rate limit)
- Sample agents with various configurations
- Temp workspace directories
- Tool mocks

**Verification:** Fixtures used across 3+ test files

---

### P2.3: Implement Conversation Caching
**Source:** Conversation cache with hit/miss tracking
**Action:**
```python
class Conversation:
    def __init__(self, cache_enabled: bool = True):
        self.caching = cache_enabled
        self._str_cache: Optional[str] = None

    def return_history_as_string(self) -> str:
        if not self.caching:
            return self._build_history_string()

        if self._str_cache is None:
            self._cache_misses += 1
            self._str_cache = self._build_history_string()
        else:
            self._cache_hits += 1
        return self._str_cache

    def add(self, role: str, content: str):
        # Invalidate cache on mutation
        self._str_cache = None
        self.conversation_history.append({"role": role, "content": content})
```

**Verification:** Cache hit rate > 80% in repeated-read workloads

---

### P2.4: Dynamic Context Window Management
**Source:** Binary search truncation for context limits
**Action:**
```python
def truncate_to_context(self, history: List[Dict], max_tokens: int) -> List[Dict]:
    # Binary search for optimal truncation point
    # Preserve recent messages, truncate older ones
    result = []
    total_tokens = 0
    for msg in reversed(history):
        msg_tokens = count_tokens(msg["content"])
        if total_tokens + msg_tokens <= max_tokens:
            result.insert(0, msg)
            total_tokens += msg_tokens
        else:
            break
    return result
```

**Verification:** Test with various context limits (4K, 16K, 128K)

---

### P2.5: Multi-Level Caching Strategy
**Source:** Conversation, agent, system info caching
**Action:**
```python
# Layer 1: Response caching (LLM calls)
@lru_cache(maxsize=1000)
def cached_complete(prompt_hash, model):
    return llm.complete(prompt)

# Layer 2: Conversation memoization
class Conversation:
    _str_cache = None  # Invalidated on mutation

# Layer 3: Agent template caching
@cache
def load_agent_template(name: str) -> Dict:
    return json.load(f"templates/{name}.json")
```

**Verification:** Performance benchmarks before/after caching

---

### P2.6: Safe Deserialization
**Source:** JSON-only loading patterns
**Action:**
```python
@staticmethod
def load_state(file_path: str) -> Dict:
    if not os.path.exists(file_path):
        raise FileNotFoundError(f"State file not found: {file_path}")

    with open(file_path, "r") as f:
        # Always use json, never yaml with unsafe loader
        return json.load(f)

    # If YAML is needed:
    # with open(file_path, "r") as f:
    #     return yaml.safe_load(f)  # safe_load, not load
```

**Verification:** Security scan for unsafe deserialization

---

### P2.7: Secret Generation Utility
**Source:** Uses `secrets` module for cryptographic RNG
**Action:**
```python
import secrets
import string

def generate_api_key(prefix: str = "sk-", length: int = 32) -> str:
    if length < 8:
        raise ValueError("Length must be at least 8 characters")
    alphabet = string.ascii_letters + string.digits
    random_part = "".join(
        secrets.choice(alphabet) for _ in range(length)
    )
    return f"{prefix}{random_part}"
```

**Verification:** Use in at least one feature (API key generation for agents)

---

## P3: Advanced Features (Post-v1.0)

### P3.1: Prompt Versioning System
**Source:** 69 prompt templates with no versioning
**Action:**
```python
class Prompt:
    def __init__(self, content: str):
        self.id = uuid.uuid4().hex
        self.content = content
        self.version = 1
        self.edit_history: List[Dict] = []

    def edit_prompt(self, new_content: str, author: str = "system"):
        self.edit_history.append({
            "version": self.version,
            "content": self.content,
            "edited_by": author,
            "timestamp": time.time()
        })
        self.version += 1
        self.content = new_content

    def rollback(self, version: int):
        for entry in self.edit_history:
            if entry["version"] == version:
                self.content = entry["content"]
                return
        raise ValueError(f"Version {version} not found")
```

**Verification:** Rollback tests pass

---

### P3.2: Swarm Orchestration Patterns
**Source:** 16+ swarm types observed
**Action:** Implement core patterns:
1. SequentialWorkflow (chain)
2. ConcurrentWorkflow (parallel)
3. MixtureOfAgents (layered aggregation)
4. HierarchicalSwarm (director pattern)

**Avoid over-engineering:** Don't create 16 types immediately. Start with 4, add based on need.

---

### P3.3: Auto-Generation with Guardrails
**Source:** AutoSwarmBuilder with hardcoded GPT-4.1
**Action:**
```python
class AutoSwarmBuilder:
    def __init__(self, boss_model: str = None, max_agents: int = 10):
        self.boss_model = boss_model or os.getenv("DEFAULT_BOSS_MODEL")
        self.max_agents = max_agents  # Prevent runaway agent creation

    def run(self, task: str):
        # Validate task complexity before spawning
        complexity = self._estimate_complexity(task)
        if complexity > self.max_agents:
            raise ValueError(f"Task complexity {complexity} exceeds max {self.max_agents}")

        # Use configurable boss model
        boss = LLMClient(model_name=self.boss_model)
        # ... rest of generation
```

**Verification:** Test with various task complexities

---

### P3.4: MCP Integration
**Source:** MCP client in swarms (42KB mcp_client_tools.py)
**Action:** If MCP becomes relevant:
```python
class MCPClient:
    async def list_tools(self) -> List[Tool]:
        # Connect to MCP server
        # Return tools in OpenAI format
        pass

    def transform_to_openai_tool(self, mcp_tool: MCPTool) -> Dict:
        return {
            "type": "function",
            "function": {
                "name": mcp_tool.name,
                "description": mcp_tool.description,
                "parameters": mcp_tool.inputSchema
            }
        }
```

**Verification:** Integration test with real MCP server

---

## Process Recommendations

### Documentation Requirements
1. README with clear getting started (< 5 min to first agent)
2. Architecture decision records (ADRs) for key choices
3. Examples directory with working code for each pattern
4. CONTRIBUTING.md with:
   - Development setup
   - Code style (with tool configs)
   - Test requirements
   - PR template

### CI/CD Requirements
1. **Must pass:**
   - pytest with coverage > 70%
   - mypy strict mode
   - black formatting check
   - ruff linting
   - Security scan (bandit)

2. **Should include:**
   - Dependency audit (safety)
   - Code complexity analysis
   - Documentation build

### Codeowners
**Action:** Create `.github/CODEOWNERS`:
```
# Default owner
* @maintainer

# Feature areas
/src/agents @agent-team
/src/tools @tools-team
/src/swarm @orchestration-team
```

---

## Summary Checklist

| Priority | Item | Status |
|----------|------|--------|
| P0 | SSL verification default to True | TODO |
| P0 | Remove side effects from import | TODO |
| P0 | Add input size limits | TODO |
| P1 | Break monolithic Agent class | TODO |
| P1 | Configure and run mypy | TODO |
| P1 | Fix black target-version | TODO |
| P1 | Standardize error handling | TODO |
| P1 | LiteLLM-style abstraction | TODO |
| P2 | Proper pytest setup | TODO |
| P2 | Shared fixtures in conftest.py | TODO |
| P2 | Conversation caching | TODO |
| P2 | Dynamic context management | TODO |
| P2 | Multi-level caching | TODO |
| P2 | Safe deserialization | TODO |
| P2 | Secret generation utility | TODO |
| P3 | Prompt versioning | TODO |
| P3 | Core swarm patterns | TODO |
| P3 | Auto-generation guardrails | TODO |

---

*Action items derived from lessons learned analysis dated 2026-03-26*
