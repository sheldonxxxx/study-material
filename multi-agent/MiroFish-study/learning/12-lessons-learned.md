# MiroFish Study: Lessons Learned

## Overview

This document synthesizes key insights from analyzing the MiroFish codebase (43K+ GitHub stars, AI-powered swarm intelligence simulation engine). These lessons inform how to build a similar project well -- and what to avoid.

---

## 1. Architecture Patterns to Emulate

### 1.1 Layered Service-Oriented Architecture

MiroFish uses a clean three-tier structure:
- **API Layer**: Route handlers (`api/*.py`) handle HTTP only
- **Service Layer**: Business logic in dedicated classes (`services/*.py`)
- **Model Layer**: Data structures and persistence (`models/*.py`)

**Why it works**: Clear separation enables testing, parallel development, and code reuse. Each service has a single responsibility.

**Example from code** (`backend/app/services/simulation_manager.py`):
```python
class SimulationManager:
    def create_simulation(self, project_id, graph_id, ...): ...
    def prepare_simulation(self, simulation_id, progress_callback=None): ...
    def get_simulation(self, simulation_id): ...
```

### 1.2 Dual Logging System

The Report Agent implements two parallel logging mechanisms:

1. **JSONL for replay** (`agent_log.jsonl`) - structured for streaming to frontend
2. **Console log for humans** (`console_log.txt`) - readable `[timestamp] LEVEL: message` format

```python
# From report_agent.py lines 84-93
log_entry = {
    "timestamp": datetime.now().isoformat(),
    "elapsed_seconds": round(self._get_elapsed_time(), 2),
    "report_id": self.report_id,
    "action": action,
    "stage": stage,
    ...
}
```

**Why it works**: JSONL enables real-time log streaming to the UI while human-readable logs simplify debugging.

### 1.3 Progress Callbacks for Async Operations

Async operations report progress via callbacks, enabling real-time UI updates:

```python
# From simulation_config_generator.py lines 278-306
def report_progress(step: int, message: str):
    nonlocal current_step
    current_step = step
    if progress_callback:
        progress_callback(step, total_steps, message)
```

**Why it works**: Frontend polling every 2-3 seconds for status updates works well for async LLM operations. The callback pattern decouples service logic from UI.

### 1.4 Temperature Annealing on LLM Retries

When LLM calls fail and retry, temperature decreases to produce more deterministic results:

```python
# From simulation_config_generator.py line 449
temperature=0.7 - (attempt * 0.1)  # Reduces from 0.7 to 0.5 to 0.3
```

**Why it works**: Initial attempts can be creative; retries become more focused. Prevents "creative wandering" on repeated failures.

### 1.5 Graceful Degradation with Rule-Based Fallbacks

When LLM generation fails, rule-based defaults ensure operation completes:

```python
# From simulation_config_generator.py lines 904-985
if entity_type in ["university", "governmentagency", "ngo"]:
    return {
        "activity_level": 0.2,
        "posts_per_hour": 0.1,
        "influence_weight": 3.0  # High influence for institutions
    }
```

**Why it works**: Users get functional results even when LLM is unavailable or rate-limited.

### 1.6 Immediate File Persistence

Long-running operations save progress incrementally:

```python
# From oasis_profile_generator.py line 452
for future in concurrent.futures.as_completed(future_to_entity):
    profiles[result_idx] = profile
    save_profiles_realtime()  # Write after each completion
```

**Why it works**: If the process crashes, partial results are preserved. Particularly valuable for generating 50+ agent profiles.

### 1.7 Batch Processing for LLM Calls

Large workloads are split into manageable chunks:

```python
# From simulation_config_generator.py line 311
num_batches = math.ceil(len(entities) / self.AGENTS_PER_BATCH)  # 15 per batch
total_steps = 3 + num_batches
```

**Why it works**: Balances LLM context window limits against API call overhead. Progress is visible per batch.

### 1.8 Smart Text Chunking with Sentence Boundaries

Text splitting prefers natural sentence boundaries:

```python
# From file_parser.py lines 486-494
separators = ['。', '！', '？', '.\n', '!\n', '?\n', '\n\n', '. ', '! ', '? ']
for sep in separators:
    last_sep = text[start:end].rfind(sep)
    if last_sep != -1 and last_sep > chunk_size * 0.3:
        end = start + last_sep + len(sep)
        break
```

**Why it works**: Chunks end at natural sentence boundaries rather than mid-sentence, improving RAG quality.

### 1.9 Multi-Level Encoding Fallback

File parsing handles encoding edge cases:

```python
# From file_parser.py lines 437-442
def _read_text_with_fallback(file_path: str) -> str:
    # 1. Try UTF-8
    # 2. charset_normalizer.from_bytes(data).best()
    # 3. chardet.detect(data)
    # 4. Fallback to UTF-8 with errors='replace'
```

**Why it works**: Chinese documents especially may have varied encodings. The fallback chain maximizes extraction success.

### 1.10 ReACT Pattern for Tool-Augmented Agents

The Report Agent uses Reasoning + Acting with reflection:

```python
# From report_agent.py lines 1220-1530
# 1. LLM responds with thought/action
# 2. Parse tool calls from response
# 3. If Final Answer detected and tool calls >= 3, output content
# 4. If tool call present, execute and inject observation
# 5. Max 5 tool calls per section, min 3 required
```

**Why it works**: Forces the agent to actually use tools before concluding, improving report depth.

---

## 2. Security Issues to Avoid

### 2.1 NO AUTHENTICATION (CRITICAL)

Every API endpoint is publicly accessible. The security analysis found:
- No JWT or session-based authentication
- No API key authentication
- No rate limiting per user/IP
- Anyone can create/delete projects, start/stop simulations

**Impact**: A production deployment would be completely open.

### 2.2 Full Tracebacks in Error Responses

```python
# From simulation.py lines 88-89
"error": str(e),
"traceback": traceback.format_exc()
```

**Impact**: Leaks internal file paths, library versions, code structure, potential exploit paths.

### 2.3 Path Traversal in Script Download

```python
# From simulation.py line 1347 - No validation of script_name
script_path = os.path.join(scripts_dir, script_name)
# An attacker could request ../../../etc/passwd
```

**Impact**: Arbitrary file read vulnerability.

### 2.4 Hardcoded SECRET_KEY

```python
# From config.py line 29
SECRET_KEY = os.environ.get('SECRET_KEY', 'mirofish-secret-key')
```

**Impact**: Ships with default secret. If deployed without proper env var, session tokens are trivially forgeable.

### 2.5 Debug Mode Defaulting to True

```python
# From config.py line 25
DEBUG = os.environ.get('FLASK_DEBUG', 'True').lower() == 'true'
```

**Impact**: Production deployments may run with debug features enabled.

### 2.6 File-Based IPC Without Authentication

The simulation IPC mechanism at `simulation_ipc.py` uses JSON files for commands:
- No authentication on IPC commands
- Any local process with filesystem access can send commands
- No verification that commands come from the Flask process

---

## 3. Code Quality Issues to Avoid

### 3.1 No Testing Infrastructure

Despite `pytest` being in `pyproject.toml`, only one manual test script exists (`test_profile_format.py`). No automated tests run in CI.

**Evidence from code quality analysis**:
- Zero unit tests
- Zero integration tests
- Zero E2E tests
- No test automation

**Impact**: Refactoring is risky. Bugs slip through.

### 3.2 No Linting/Formatting

- No ESLint, Prettier for frontend
- No ruff, black, isort for backend
- Chinese comments mixed with English throughout
- Inconsistent code style will emerge with multiple contributors

### 3.3 TypeScript Listed But Not Used

The `frontend/package.json` lists TypeScript as a dependency, but:
- No `tsconfig.json` found
- No `.d.ts` type definition files
- Code uses `.js` extension with no type annotations

### 3.4 Monolithic Files

| File | Lines |
|------|-------|
| `simulation.py` (API) | 2,712 |
| `simulation_runner.py` | 1,764 |
| `report_agent.py` | 99KB |
| `Step4Report.vue` | 145KB |

**Impact**: Unmaintainable. Difficult to understand, test, or review.

### 3.5 No CI Quality Gates

GitHub Actions only runs Docker image build on tags. No:
- Linting checks
- Type checking
- Test execution
- Security scanning

---

## 4. What Surprised Me

### 4.1 Single Maintainer, Large Star Count

Despite 43K stars and 6K forks:
- 99.5% of commits from one person (666ghj)
- No formal governance (no CODEOWNERS, MAINTAINERS, CONTRIBUTING.md)
- Only one GitHub Actions workflow (Docker build only)

**Implication**: High star counts do not indicate healthy community governance.

### 4.2 Simple Deployment Architecture

Single Docker image contains both frontend AND backend:
```dockerfile
# Entrypoint uses npm run dev (concurrently runs both)
ENTRYPOINT ["npm", "run", "dev"]
```

No Kubernetes, no service mesh, no complex orchestration. **This simplicity is a feature**, not a limitation.

### 4.3 uv for Python Package Management

Uses `uv` (astral-sh) instead of pip:
- `pyproject.toml` for dependencies
- `uv.lock` for reproducible installs
- Much faster than pip

**Implication**: Modern Python tooling is production-ready and worth adopting.

### 4.4 Platform-Specific Code Duplication

Reddit and Twitter simulation scripts (`run_reddit_simulation.py`, `run_twitter_simulation.py`) are nearly identical with only minor variations. The analysis noted 7 duplicate code blocks including `setup_oasis_logging`, `UnicodeFormatter`, `IPCHandler`.

**Implication**: Even a single developer with 99.5% of commits produces duplicated code. Architecture decisions about shared code matter.

### 4.5 No Database = No Transactions

The project uses file-based JSON persistence for projects and tasks:
- No database means no ACID transactions
- In-memory TaskManager loses tasks on server restart
- No concurrency control on project files

**Implication**: This is a deliberate tradeoff for simplicity. For a v0.1.x project, this is acceptable. Production would need proper database.

### 4.6 Windows UTF-8 Encoding Monkey Patch

The backend includes runtime patching of Python's built-in `open`:

```python
# From simulation_runner.py lines 150-155
def _utf8_open(file, mode='r', ...):
    if encoding is None and 'b' not in mode:
        encoding = 'utf-8'
    return _original_open(file, mode, ...)
builtins.open = _utf8_open
```

**Why this exists**: OASIS library doesn't specify encoding. Windows default encoding causes failures. The monkey patch is a clever workaround that avoids forking OASIS.

---

## 5. Patterns Worth Noting

### 5.1 Consistent API Response Envelope

All backend endpoints return:
```json
{
  "success": true,
  "data": { ... },
  "error": null
}
```

### 5.2 Dataclass Value Objects with Serialization

Rich domain objects with `to_dict()` methods:
```python
@dataclass
class SimulationState:
    def to_dict(self) -> Dict[str, Any]: ...
    def to_simple_dict(self) -> Dict[str, Any]: ...
```

### 5.3 Retry with Exponential Backoff

Decorator-based retry for transient failures:
```python
@retry_with_backoff(max_retries=3, initial_delay=1.0, backoff_factor=2.0, jitter=True)
```

### 5.4 Context Reuse for Consistency

When generating configs in batches, the same context object is reused:
```python
# From simulation_config_generator.py line 320
context = build_context(...)
for batch in batches:
    generate_batch(context, batch)  # Same context ensures consistency
```

### 5.5 Type Alias Mapping

Flexible entity type matching with fallbacks:
```python
type_aliases = {
    "official": ["official", "university", "governmentagency", "government"],
    "student": ["student", "person"],
}
```

---

## 6. Summary Assessment

| Dimension | MiroFish Score | Implication |
|-----------|---------------|-------------|
| Architecture | 7/10 | Clean layered design, but monolithic files hurt |
| Security | 2/10 | No auth, traceback leaks, path traversal |
| Testing | 2/10 | No automated tests |
| Code Quality | 4/10 | Good error handling, no linting |
| Documentation | 5/10 | README exists, no internal docs |
| DevOps | 5/10 | Docker works, no CI beyond build |

**Overall**: A functional prototype that achieved significant user interest but lacks production-readiness in security, testing, and code quality infrastructure. The architecture decisions (layered services, progress callbacks, graceful degradation) are sound and worth emulating. The security and testing gaps are critical lessons on what NOT to ship with.
