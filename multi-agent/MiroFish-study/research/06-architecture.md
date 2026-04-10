# MiroFish Architecture Analysis

## 1. Architectural Pattern

**Primary Pattern: Layered Architecture with Service-Oriented Design**

The MiroFish application follows a **three-tier layered architecture**:

```
+------------------+
|   Frontend       |  Vue.js 3 + Vite (Port 3000)
|   (Presentation) |
+--------+---------+
         |
         | REST API (HTTP/JSON)
         |
+--------+---------+
|   Backend        |  Flask + Python (Port 5001)
|   (Business)     |
+--------+---------+
         |
         | Service Layer
         |
+--------+---------+
|   External       |  Zep (Knowledge Graph), LLM (OpenAI-compatible)
|   Services       |
+------------------+
```

### Backend Layer Structure

| Layer | Components | Responsibility |
|-------|------------|-----------------|
| **API Layer** | `api/graph.py`, `api/simulation.py`, `api/report.py` | HTTP request handling, route definitions, request validation |
| **Service Layer** | `services/*.py` | Business logic, orchestration, state management |
| **Model Layer** | `models/project.py`, `models/task.py` | Data structures, persistence |
| **Utility Layer** | `utils/*.py` | Shared functionality (LLM client, logging, retry) |

### Frontend Layer Structure

| Layer | Components | Responsibility |
|-------|------------|-----------------|
| **API Client** | `api/*.js` | HTTP requests to backend |
| **Components** | `components/*.vue` | Reusable UI elements |
| **Views** | `views/*.vue` | Page-level components |
| **Store** | `store/` | State management (Pinia assumed) |
| **Router** | `router/index.js` | Client-side routing |

---

## 2. Key Abstractions and Interfaces

### 2.1 Backend Service Abstractions

**SimulationManager** (`services/simulation_manager.py`)
- Responsibility: Orchestrates simulation lifecycle (create, prepare, track state)
- Key methods: `create_simulation()`, `prepare_simulation()`, `get_simulation()`
- State: Persisted to `state.json` in simulation directory

**SimulationRunner** (`services/simulation_runner.py`)
- Responsibility: Executes OASIS simulation in background subprocess
- Key methods: `start_simulation()`, `stop_simulation()`, `get_run_state()`
- Communication: IPC via JSON files (`run_state.json`, `actions.jsonl`)

**GraphBuilderService** (`services/graph_builder.py`)
- Responsibility: Constructs Zep knowledge graphs from documents
- Key methods: `create_graph()`, `set_ontology()`, `add_text_batches()`, `get_graph_data()`

**ZepEntityReader** (`services/zep_entity_reader.py`)
- Responsibility: Reads and filters entities from Zep graphs
- Key abstractions: `EntityNode`, `FilteredEntities` dataclasses

**OntologyGenerator** (`services/ontology_generator.py`)
- Responsibility: Uses LLM to generate ontology from documents
- Pattern: Prompt-based generation with structured output

**ReportAgent** (`services/report_agent.py`)
- Responsibility: Generates analysis reports using LLM + Zep retrieval
- Pattern: Tool-augmented agent with search capabilities

### 2.2 Data Models

```python
# Project State (persisted)
@dataclass
class Project:
    project_id: str
    name: str
    status: ProjectStatus
    ontology: Optional[Dict]
    graph_id: Optional[str]
    simulation_requirement: Optional[str]

# Simulation State (persisted)
@dataclass
class SimulationState:
    simulation_id: str
    project_id: str
    graph_id: str
    status: SimulationStatus
    entities_count: int
    config_generated: bool

# Runtime State (transient, via IPC)
@dataclass
class SimulationRunState:
    runner_status: RunnerStatus
    current_round: int
    twitter_running: bool
    reddit_running: bool
    twitter_actions_count: int
    reddit_actions_count: int
```

### 2.3 API Contract Pattern

All API responses follow a consistent envelope:

```json
{
  "success": true,
  "data": { ... },
  "error": null
}
```

---

## 3. Component Communication

### 3.1 Frontend to Backend

**Technology**: Axios HTTP client with interceptors

**Base Configuration** (`frontend/src/api/index.js`):
```javascript
const service = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || 'http://localhost:5001',
  timeout: 300000  // 5 minutes for long operations
})
```

**Communication Patterns**:

| Pattern | Use Case | Implementation |
|---------|----------|----------------|
| **Request-Response** | Single-shot operations | Direct axios call |
| **Polling** | Status updates, progress | `setInterval` with GET requests |
| **Retry with Backoff** | Transient failures | `requestWithRetry()` wrapper |

### 3.2 Backend Internal Communication

**Synchronous**:
- API routes call service methods directly
- Services use utility classes (LLMClient, Logger)

**Asynchronous**:
- Background tasks run in `threading.Thread` (daemon threads)
- Progress reported via callback functions
- State persisted to filesystem

**Inter-Process Communication (Simulation Runner)**:

```
SimulationRunner (Flask process)
         |
         | subprocess.Popen
         v
OASIS Simulation Script (separate process)
         |
         | JSON files
         v
run_state.json, actions.jsonl
```

**IPC File Structure**:
```
backend/uploads/simulations/{simulation_id}/
  state.json              # SimulationState (created by SimulationManager)
  simulation_config.json  # Configuration (created by SimulationConfigGenerator)
  reddit_profiles.json    # Agent profiles
  twitter_profiles.csv    # Agent profiles
  run_state.json         # Runtime state (written by runner, read by Flask)
  actions.jsonl          # Action log (appended by runner)
```

### 3.3 External Service Communication

**Zep (Knowledge Graph)**:
- SDK: `zep-cloud` Python client
- Operations: Create graph, add episodes, query nodes/edges
- Authentication: API key via `ZEP_API_KEY`

**LLM Provider**:
- Abstraction: `LLMClient` class in `utils/llm_client.py`
- Interface: OpenAI-compatible API
- Configuration: `LLM_API_KEY`, `LLM_BASE_URL`, `LLM_MODEL_NAME`
- Features: Chat, JSON mode, retry with exponential backoff

---

## 4. Design Patterns in Use

### 4.1 Service Layer Pattern

Each service is a focused class with clear responsibility:

```
SimulationManager  - Simulation lifecycle
SimulationRunner  - Execution control
GraphBuilderService - Graph construction
ZepEntityReader   - Entity retrieval
```

### 4.2 Factory Pattern

**LLMClient** (`utils/llm_client.py`):
```python
class LLMClient:
    def chat(self, messages, ...) -> str: ...
    def chat_json(self, messages, ...) -> Dict: ...
```

Provides factory methods for different response types.

### 4.3 Strategy Pattern

**Platform-specific simulation**:
- Twitter and Reddit have distinct action spaces
- Config generator creates platform-specific profiles
- Runner executes platform-specific scripts

```python
OASIS_TWITTER_ACTIONS = ['CREATE_POST', 'LIKE_POST', 'REPOST', ...]
OASIS_REDDIT_ACTIONS = ['LIKE_POST', 'CREATE_COMMENT', ...]
```

### 4.4 Observer Pattern

**Progress callbacks** for async operations:

```python
def progress_callback(stage, progress, message, **kwargs):
    task_manager.update_task(task_id, progress=current_progress, ...)

manager.prepare_simulation(..., progress_callback=progress_callback)
```

### 4.5 Manager Pattern

Static manager classes for singleton-like access:

```python
class ProjectManager:
    @classmethod
    def get_project(cls, project_id): ...
    @classmethod
    def save_project(cls, project): ...
```

### 4.6 Dataclass Value Objects

Rich domain objects with methods:

```python
@dataclass
class SimulationState:
    simulation_id: str
    # ... fields ...

    def to_dict(self) -> Dict[str, Any]: ...
    def to_simple_dict(self) -> Dict[str, Any]: ...
```

### 4.7 Retry Pattern

Exponential backoff for transient failures:

```python
# Backend
@retry(attempts=3, delay=1, backoff=2)
def some_unstable_operation(): ...

# Frontend
export const requestWithRetry = async (requestFn, maxRetries=3, delay=1000) => {
  for (let i = 0; i < maxRetries; i++) {
    try { return await requestFn(); }
    catch (error) { await delay * Math.pow(2, i); }
  }
}
```

---

## 5. Major Modules and Responsibilities

### 5.1 Backend Modules

| Module | Files | Responsibility |
|--------|-------|----------------|
| **API Routes** | `api/graph.py`, `api/simulation.py`, `api/report.py` | HTTP endpoints, request parsing, response formatting |
| **Simulation** | `simulation_manager.py`, `simulation_runner.py`, `simulation_config_generator.py` | Full simulation lifecycle management |
| **Graph/RAG** | `graph_builder.py`, `zep_entity_reader.py`, `zep_tools.py`, `ontology_generator.py` | Knowledge graph construction and querying |
| **Profiles** | `oasis_profile_generator.py` | Agent persona generation |
| **Reporting** | `report_agent.py`, `zep_graph_memory_updater.py` | Report generation with RAG |
| **IPC** | `simulation_ipc.py` | Inter-process communication for simulation control |
| **Persistence** | `models/project.py`, `models/task.py` | File-based state storage |

### 5.2 Frontend Modules

| Module | Files | Responsibility |
|--------|-------|----------------|
| **API Client** | `api/index.js`, `api/graph.js`, `api/simulation.js`, `api/report.js` | Backend communication |
| **Components** | `components/Step*.vue` | Step-based workflow UI |
| **Views** | `views/*.vue` | Page components |
| **Router** | `router/index.js` | Route definitions |

---

## 6. Frontend-Backend Interaction

### 6.1 Workflow Sequence

```
1. Upload Documents
   Frontend: POST /api/graph/ontology/generate (multipart/form-data)
   Backend:  FileParser.extract_text() -> OntologyGenerator.generate()
   Response: { project_id, ontology, analysis_summary }

2. Build Graph
   Frontend: POST /api/graph/build { project_id }
   Backend:  GraphBuilderService.create_graph() + add_text_batches()
   Status:   GET /api/graph/task/{task_id}

3. Create Simulation
   Frontend: POST /api/simulation/create { project_id }
   Backend:  SimulationManager.create_simulation()
   Response: { simulation_id }

4. Prepare Simulation
   Frontend: POST /api/simulation/prepare { simulation_id }
   Backend:  ZepEntityReader -> OasisProfileGenerator -> SimulationConfigGenerator
   Status:   POST /api/simulation/prepare/status

5. Run Simulation
   Frontend: POST /api/simulation/start { simulation_id }
   Backend:  SimulationRunner.start_simulation() (spawns subprocess)
   Status:   GET /api/simulation/{id}/run-status (polling)

6. Generate Report
   Frontend: POST /api/report/generate { simulation_id }
   Backend:  ReportAgent.generate_report() (LLM + Zep retrieval)
   Status:   POST /api/report/generate/status
```

### 6.2 Real-time Updates

**Polling Strategy**:
- Frontend polls `GET /api/simulation/{id}/run-status` every 2-5 seconds
- Each poll returns current round, action counts, platform status
- Actions retrieved via `GET /api/simulation/{id}/actions`

**File-based State Sharing**:
```
Flask Process writes -> run_state.json <- Simulation Process reads
                   writes -> actions.jsonl (append-only)
```

---

## 7. Data Flow Diagram

```
User Upload
    |
    v
Frontend (Vue.js) --> REST API --> Flask (API Layer)
                                        |
                    +-------------------+-------------------+
                    |                   |                   |
                    v                   v                   v
            GraphBuilderService   SimulationManager   ReportAgent
                    |                   |                   |
                    v                   v                   v
            Zep Cloud API        Subprocess + IPC     LLM + Zep Retrieval
                    |                   |                   |
                    v                   v                   v
            Knowledge Graph       SQLite + Files       Markdown Report
```

---

## 8. State Persistence

| State Type | Storage | Access |
|------------|---------|--------|
| Project metadata | `project.json` (file) | ProjectManager |
| Simulation config | `state.json`, `simulation_config.json` | SimulationManager |
| Simulation runtime | `run_state.json` (IPC) | SimulationRunner |
| Action log | `actions.jsonl` (append) | SimulationRunner |
| Agent profiles | `reddit_profiles.json`, `twitter_profiles.csv` | SimulationRunner |
| Task progress | In-memory + file | TaskManager |
| Report | `report.json` + `report.md` | ReportManager |

---

## 9. Configuration Management

**Environment Variables** (`.env`):
```
LLM_API_KEY          # LLM provider API key
LLM_BASE_URL         # LLM API endpoint
LLM_MODEL_NAME        # Model identifier (e.g., qwen-plus)
ZEP_API_KEY          # Zep knowledge graph API key
FLASK_DEBUG           # Debug mode toggle
```

**Application Config** (`app/config.py`):
- File upload limits
- Chunk sizes for text processing
- OASIS platform action definitions
- Report agent parameters

---

## 10. Key Design Decisions

1. **File-based IPC over Message Queue**: Chosen for simplicity; append-only log files work well for action tracking

2. **Daemon Threads for Background Tasks**: Avoids full multiprocessing complexity; cleanup registered via `atexit`

3. **Dataclasses for Domain Objects**: Python dataclasses provide clean, typed data structures with serialization methods

4. **Service Layer over Direct API Logic**: Business logic separated from HTTP handling enables testability

5. **Polling over WebSocket**: Simple long-polling for status updates; sufficient for current requirements

6. **LLM Client Abstraction**: OpenAI-compatible interface allows switching providers without code changes

---

## 11. Summary

MiroFish implements a **layered, service-oriented architecture** with clear separation between presentation (Vue.js), business logic (Flask services), and data (Zep + filesystem). The design emphasizes:

- **Async task handling** via daemon threads with progress callbacks
- **IPC via filesystem** for simulation subprocess control
- **Service abstractions** for business logic encapsulation
- **RAG-powered reporting** combining LLM with knowledge graph retrieval

The architecture is well-suited for CPU-intensive simulation workloads with real-time monitoring requirements.
