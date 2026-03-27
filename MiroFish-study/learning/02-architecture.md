# MiroFish Architecture Analysis

## Architectural Overview

MiroFish implements a **layered, service-oriented architecture** with clear separation between presentation (Vue.js), business logic (Flask services), and data (Zep + filesystem).

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

## Layer Responsibilities

### Backend Layer Structure

| Layer | Components | Responsibility |
|-------|------------|-----------------|
| **API Layer** | `api/graph.py`, `api/simulation.py`, `api/report.py` | HTTP request handling, route definitions, request validation |
| **Service Layer** | `services/*.py` | Business logic orchestration, state management, external service integration |
| **Model Layer** | `models/project.py`, `models/task.py` | Data structures, persistence |
| **Utility Layer** | `utils/*.py` | Shared functionality (LLM client, logging, retry) |

### Frontend Layer Structure

| Layer | Components | Responsibility |
|-------|------------|-----------------|
| **API Client** | `api/*.js` | HTTP requests to backend |
| **Components** | `components/*.vue` | Reusable UI elements |
| **Views** | `views/*.vue` | Page-level components |
| **Store** | `store/` | State management |
| **Router** | `router/index.js` | Client-side routing |

## Key Architectural Decisions

### 1. File-based IPC over Message Queue

**Decision:** Chosen for simplicity; append-only log files work well for action tracking.

**Implementation:** Simulation runner communicates with subprocess via JSON files in the simulation directory:
- `run_state.json` - Runtime state written by runner, read by Flask
- `actions.jsonl` - Append-only action log
- `ipc_commands/` and `ipc_responses/` directories for request-response pattern

**File:** `backend/app/services/simulation_ipc.py`

```python
class SimulationIPCClient:
    def send_command(self, command_type: CommandType, args: Dict[str, Any], ...) -> IPCResponse:
        # Write command to ipc_commands/{command_id}.json
        # Poll ipc_responses/{command_id}.json until response or timeout
```

### 2. Daemon Threads for Background Tasks

**Decision:** Avoids full multiprocessing complexity; cleanup registered via `atexit`.

**Implementation:** Background operations run in `threading.Thread` with `daemon=True`.

**File:** `backend/app/services/graph_builder.py`

```python
def build_graph_async(self, text: str, ontology: Dict, ...):
    thread = threading.Thread(
        target=self._build_graph_worker,
        args=(task_id, text, ontology, ...)
    )
    thread.daemon = True
    thread.start()
```

### 3. Dataclasses for Domain Objects

**Decision:** Python dataclasses provide clean, typed data structures with serialization methods.

**File:** `backend/app/models/project.py`

```python
@dataclass
class Project:
    project_id: str
    name: str
    status: ProjectStatus
    # ... fields ...

    def to_dict(self) -> Dict[str, Any]: ...
    @classmethod
    def from_dict(cls, data: Dict[str, Any]) -> 'Project': ...
```

### 4. Service Layer over Direct API Logic

**Decision:** Business logic separated from HTTP handling enables testability.

**Evidence:** Each API route handler delegates to a service class:

| API Route | Service |
|-----------|---------|
| `api/graph.py` | `GraphBuilderService` |
| `api/simulation.py` | `SimulationManager`, `SimulationRunner` |
| `api/report.py` | `ReportAgent`, `ReportManager` |

### 5. Polling over WebSocket

**Decision:** Simple long-polling for status updates; sufficient for current requirements.

**Implementation:** Frontend polls `GET /api/simulation/{id}/run-status` every 2-5 seconds.

### 6. LLM Client Abstraction

**Decision:** OpenAI-compatible interface allows switching providers without code changes.

**File:** `backend/app/utils/llm_client.py`

```python
class LLMClient:
    def chat(self, messages: List[Dict[str, str]], ...) -> str:
        response = self.client.chat.completions.create(
            model=self.model_name,
            messages=messages,
            ...
        )
        return response.choices[0].message.content
```

## Module Responsibilities

### Backend Modules

| Module | Files | Responsibility |
|--------|-------|-----------------|
| **API Routes** | `api/graph.py`, `api/simulation.py`, `api/report.py` | HTTP endpoints, request parsing, response formatting |
| **Simulation** | `simulation_manager.py`, `simulation_runner.py`, `simulation_config_generator.py` | Full simulation lifecycle management |
| **Graph/RAG** | `graph_builder.py`, `zep_entity_reader.py`, `zep_tools.py`, `ontology_generator.py` | Knowledge graph construction and querying |
| **Profiles** | `oasis_profile_generator.py` | Agent persona generation |
| **Reporting** | `report_agent.py`, `zep_graph_memory_updater.py` | Report generation with RAG |
| **IPC** | `simulation_ipc.py` | Inter-process communication for simulation control |
| **Persistence** | `models/project.py`, `models/task.py` | File-based state storage |

### Frontend Modules

| Module | Files | Responsibility |
|--------|-------|-----------------|
| **API Client** | `api/index.js`, `api/graph.js`, `api/simulation.js`, `api/report.js` | Backend communication |
| **Components** | `components/Step*.vue` | Step-based workflow UI |
| **Views** | `views/*.vue` | Page components |
| **Router** | `router/index.js` | Route definitions |

## Communication Patterns

### Frontend to Backend Communication

**Technology:** Axios HTTP client with interceptors

**File:** `frontend/src/api/index.js`

```javascript
const service = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || 'http://localhost:5001',
  timeout: 300000  // 5 minutes for long operations
})
```

**Patterns:**
| Pattern | Use Case | Implementation |
|---------|----------|----------------|
| Request-Response | Single-shot operations | Direct axios call |
| Polling | Status updates, progress | `setInterval` with GET requests |
| Retry with Backoff | Transient failures | `requestWithRetry()` wrapper |

### Backend Internal Communication

**Synchronous:**
- API routes call service methods directly
- Services use utility classes (LLMClient, Logger)

**Asynchronous:**
- Background tasks run in `threading.Thread` (daemon threads)
- Progress reported via callback functions
- State persisted to filesystem

**Inter-Process Communication (Simulation Runner):**

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

### External Service Communication

**Zep (Knowledge Graph):**
- SDK: `zep-cloud` Python client
- Operations: Create graph, add episodes, query nodes/edges
- Authentication: API key via `ZEP_API_KEY`

**LLM Provider:**
- Abstraction: `LLMClient` class in `utils/llm_client.py`
- Interface: OpenAI-compatible API
- Configuration: `LLM_API_KEY`, `LLM_BASE_URL`, `LLM_MODEL_NAME`

## State Persistence

| State Type | Storage | Access |
|------------|---------|--------|
| Project metadata | `project.json` (file) | ProjectManager |
| Simulation config | `state.json`, `simulation_config.json` | SimulationManager |
| Simulation runtime | `run_state.json` (IPC) | SimulationRunner |
| Action log | `actions.jsonl` (append) | SimulationRunner |
| Agent profiles | `reddit_profiles.json`, `twitter_profiles.csv` | SimulationRunner |
| Task progress | In-memory + file | TaskManager |
| Report | `report.json` + `report.md` | ReportManager |

## Data Flow

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

## Workflow Sequence

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

## Configuration Management

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
