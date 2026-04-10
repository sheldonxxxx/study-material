# My Action Items for Building a Similar Project

**Project:** Planning derived from CAMEL-Oasis analysis
**Date:** 2026-03-27
**Purpose:** Concrete next steps for building an agent-based social simulation platform

---

## Priority 1: Foundation (Do First)

### 1.1 Set Up Project Infrastructure

- [ ] Initialize Python project with Poetry
- [ ] Configure pre-commit hooks (license header, ruff, isort, yamllint, mdformat)
- [ ] Set up GitHub Actions CI pipeline
- [ ] Add mypy to pre-commit and CI from day one (avoid OASIS's type safety gap)
- [ ] Create .env.example with required API keys

**Why:** Establish quality standards before writing code. OASIS added these later with inconsistent enforcement.

---

### 1.2 Design Architecture Before Implementation

- [ ] Document layered architecture: Orchestration > Agent > Platform > Data
- [ ] Define interface contracts between layers (Protocol classes)
- [ ] Choose async-first design (asyncio)
- [ ] Plan semaphore-based concurrency control (default: 128)

**Why:** OASIS's clean architecture emerged organically. Starting with explicit design prevents refactoring later.

---

### 1.3 Plan State Management Strategy

- [ ] Avoid module-level global state (use class instances or dependency injection)
- [ ] If globals necessary, implement `reset_globals()` and call it in environment reset
- [ ] Design database session/connection injection pattern

**Why:** OASIS's global state in recsys causes memory leaks and test difficulties. This is the hardest debt to pay back.

---

## Priority 2: Core Features (Implement Carefully)

### 2.1 Agent System

- [ ] Extend CAMEL ChatAgent for core functionality (emulate OASIS pattern)
- [ ] Implement SocialAgent with profile-based system prompts
- [ ] Build action tool system using CAMEL's FunctionTool
- [ ] Design action filtering per platform (Twitter vs Reddit subset)
- [ ] Implement AgentGraph with pluggable backend (igraph for memory, Neo4j for scale)

**Code reference from OASIS:**
```python
class SocialAgent(ChatAgent):
    def __init__(self, agent_id, user_info, model, available_actions, tools, ...):
        self.action_tools = self.env.action.get_openai_function_list()
        all_tools = (tools or []) + (self.action_tools or [])
        super().__init__(system_message=system_message, model=model, tools=all_tools)
```

---

### 2.2 Platform Layer

- [ ] Implement channel-based IPC (Actor pattern with async queues)
- [ ] Design action routing via `getattr(self, action.value)` pattern
- [ ] Create platform abstraction (base class with Twitter/Reddit specializations)
- [ ] Use Strategy pattern for platform differences (avoid if-else chains)

**Why:** OASIS's platform if-else chains are scattered throughout 1600+ lines. Extract to strategy.

---

### 2.3 Database Design

- [ ] Use SQLite with proper schema files
- [ ] **Enable foreign keys explicitly:** `PRAGMA foreign_keys = ON`
- [ ] Add indexes on frequently queried columns:
  - `post(user_id, created_at)`
  - `follow(follower_id, followee_id)`
  - `trace(user_id, action, created_at)`
- [ ] Implement basic migration system (even if simple)
- [ ] Use parameterized queries exclusively (prevent SQL injection)

**Why:** OASIS has no indexes and foreign keys disabled. Query performance will degrade.

---

### 2.4 Recommendation System

- [ ] Implement pluggable strategy pattern for recsys
- [ ] Start with Random and Hot-score algorithms
- [ ] Add embedding-based personalization later
- [ ] **Critical:** Use class-based state, not module globals
- [ ] Implement `reset_globals()` and call in environment reset
- [ ] Add memory management (clear caches at appropriate intervals)

**Why:** OASIS's recsys globals cause unbounded memory growth.

---

## Priority 3: Scale Considerations

### 3.1 Million-Agent Support

- [ ] Design batch SQL operations for agent creation
- [ ] Implement activity thresholding (not all agents act every timestep)
- [ ] Plan for graph edge bypass at extreme scale (document limitation)
- [ ] Use asyncio.Semaphore to limit concurrent LLM calls

**OASIS pattern to emulate:**
```python
for agent in agent_graph:
    if agent.user_info.is_controllable is False:
        agent_ac_prob = random.random()
        threshold = agent.user_info.profile['other_info']['active_threshold'][...]
        if agent_ac_prob < threshold:
            tasks.append(agent.perform_action_by_llm())
```

---

### 3.2 Performance Optimizations

- [ ] Use `executemany()` for batch inserts
- [ ] Add LIMIT clauses to unbounded queries
- [ ] Implement coarse filtering for recommendation candidates (OASIS uses 4000)
- [ ] Consider PRAGMA settings carefully (synchronous=NORMAL, not OFF)

**Why:** OASIS disabled synchronous writes for speed, risking corruption.

---

## Priority 4: Code Quality

### 4.1 Error Handling

- [ ] Create custom exception hierarchy:
  - `SimulationError` (base)
  - `PlatformError`, `AgentError`, `DatabaseError` (specific)
- [ ] Catch specific exceptions, not `Exception`
- [ ] Log errors with structured data (include error codes, context)
- [ ] Return error information in response dicts without exposing internals

**Why:** OASIS has 65 broad exception blocks that mask errors.

---

### 4.2 Type Safety

- [ ] Add mypy to CI: `mypy src/ --ignore-missing-imports`
- [ ] Enable strict checking gradually as codebase matures
- [ ] Use `TYPE_CHECKING` for circular imports
- [ ] Consider TypedDict, Protocol, Literal types for complex structures

**Why:** OASIS has type hints but no enforcement. Bugs caught only at runtime.

---

### 4.3 Testing

- [ ] Set up pytest with pytest-asyncio for async tests
- [ ] Create MockChannel for platform communication testing
- [ ] Write integration tests with actual SQLite database
- [ ] Add unit tests for pure functions
- [ ] Test failure cases (duplicate records, not found, etc.)

**OASIS pattern to emulate:**
```python
class MockChannel:
    def __init__(self):
        self.call_count = 0
        self.messages = []

    async def receive_from(self):
        if self.call_count == 0:
            self.call_count += 1
            return ("id_", (1, ("alice0101", "Alice", "A girl."), "sign_up"))
        else:
            return ("id_", (None, None, "exit"))

    async def send_to(self, message):
        self.messages.append(message)
```

---

## Priority 5: Research Features (If Needed)

### 5.1 Interview Action

- [ ] If implementing, make it available to LLM-driven agents (not just ManualAction)
- [ ] Design dedicated interview table with schema
- [ ] Implement response validation

**Why:** OASIS's Interview requires ManualAction, limiting autonomous research.

---

### 5.2 Content Moderation

- [ ] If implementing reporting, add automatic moderation actions
- [ ] Define threshold-based content hiding/deletion
- [ ] Create report categories for analysis

**Why:** OASIS reports are stored but never act on content.

---

### 5.3 Profile Generation

- [ ] Use demographic distributions (age, gender, MBTI) for realism
- [ ] Consider RAG-based enhancement with vector database
- [ ] Implement BA network model for follow relationships
- [ ] Use ThreadPoolExecutor for parallel generation at scale

**OASIS pattern to emulate:**
```python
ages = ["13-17", "18-24", "25-34", "35-49", "50+"]
p_ages = [0.066, 0.171, 0.385, 0.207, 0.171]

mbtis = ["ISTJ", "ISFJ", "INFJ", "INTJ", ...]  # 16 types
p_mbti = [0.12625, 0.11625, 0.02125, ...]  # realistic distribution
```

---

## Priority 6: Documentation and Community

### 6.1 Documentation

- [ ] Create comprehensive CONTRIBUTING.md
- [ ] Set up Mintlify or similar for API docs
- [ ] Document docstring guidelines (Google Python Style Guide)
- [ ] Create example scripts for each feature

**Why:** OASIS has excellent CONTRIBUTING.md and issue templates.

---

### 6.2 Community Infrastructure

- [ ] Set up GitHub issue templates (Bug Report, Feature Request, Questions)
- [ ] Create PR template with checklist
- [ ] Define PR labeling convention (feat, fix, docs, style, refactor, test, chore)
- [ ] Plan communication channels (Discord, meetings)

**Why:** OASIS's community infrastructure is thorough and enables contribution.

---

## Implementation Checklist Summary

| Category | Action Item | Priority | Status |
|----------|-------------|----------|--------|
| Infrastructure | Add mypy to CI | P1 | Not started |
| Infrastructure | Pre-commit hooks | P1 | Not started |
| Architecture | Define layer interfaces | P1 | Not started |
| State | Avoid global state | P1 | Not started |
| Database | Enable foreign keys | P2 | Not started |
| Database | Add indexes | P2 | Not started |
| Error Handling | Custom exceptions | P4 | Not started |
| Testing | pytest + MockChannel | P4 | Not started |
| Recsys | Class-based state | P2 | Not started |
| Scale | Activity thresholding | P3 | Not started |

---

## Anti-Patterns to Avoid

1. **Do not disable SQLite synchronous** - Risk of corruption
2. **Do not leave commented debug code** - Remove before committing
3. **Do not use broad exception catching** - Catch specific exceptions
4. **Do not scatter platform conditionals** - Use strategy pattern
5. **Do not let globals accumulate** - Implement reset functions
6. **Do not skip database indexes** - Add early, optimize later

---

## Reference Architecture

```
User Code
    |
    v
generate_*_agent_graph() --> AgentGraph with profiles
    |
    v
oasis.make() --> OasisEnv (creates Platform internally)
    |
    v
env.reset() --> Platform.running() task + agent registration
    |
    v
env.step(actions) --> Execute all actions concurrently
    |
    +--> Platform (handles DB, recsys, actions via channel)
    +--> AgentGraph (manages agent execution)
    +--> Clock (simulation time)
```

---

*Derived from analysis of CAMEL-Oasis project (https://github.com/camel-ai/oasis)*
