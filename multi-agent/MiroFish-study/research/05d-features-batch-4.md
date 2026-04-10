# Feature Batch 4 Deep Dive

**Features Covered:** Simulation History & Project Management, Platform-Specific Simulations (Reddit, Twitter), Data Input Processing
**Date:** 2026-03-27
**Repo:** /Users/sheldon/Documents/claw/reference/MiroFish

---

## Feature 10: Simulation History & Project Management

### Core Implementation Files

| File | Size | Purpose |
|------|------|---------|
| `frontend/src/components/HistoryDatabase.vue` | 34KB | Vue component for displaying simulation history |
| `backend/app/models/project.py` | 9KB | Project model with ProjectManager for persistence |
| `backend/app/models/task.py` | 6KB | Task model with TaskManager for async task tracking |

### Architecture Overview

The History & Project Management system uses a **file-based JSON persistence** approach rather than a traditional database.

#### Project Model (`project.py`)

**ProjectStatus Enum:**
```python
class ProjectStatus(str, Enum):
    CREATED = "created"              # Files uploaded
    ONTOLOGY_GENERATED = "ontology_generated"  # Ontology generated
    GRAPH_BUILDING = "graph_building"    # Graph building in progress
    GRAPH_COMPLETED = "graph_completed"  # Graph complete
    FAILED = "failed"                # Failed
```

**Project Dataclass:**
```python
@dataclass
class Project:
    project_id: str
    name: str
    status: ProjectStatus
    created_at: str
    updated_at: str

    # File information
    files: List[Dict[str, str]] = field(default_factory=list)
    total_text_length: int = 0

    # Ontology information
    ontology: Optional[Dict[str, Any]] = None
    analysis_summary: Optional[str] = None

    # Graph information
    graph_id: Optional[str] = None
    graph_build_task_id: Optional[str] = None

    # Configuration
    simulation_requirement: Optional[str] = None
    chunk_size: int = 500
    chunk_overlap: int = 50
```

**ProjectManager Key Methods:**
- `create_project(name)` - Creates project with directory structure
- `save_project(project)` - Persists to `project.json`
- `get_project(project_id)` - Loads from `project.json`
- `list_projects(limit)` - Lists all projects, sorted by creation time
- `delete_project(project_id)` - Removes project directory via `shutil.rmtree`
- `save_file_to_project()` - Stores uploaded files with UUID filenames
- `save_extracted_text()` / `get_extracted_text()` - Text extraction caching

**Storage Structure:**
```
backend/uploads/projects/
  proj_{id}/
    project.json        # Metadata
    extracted_text.txt  # Cached extracted text
    files/              # Original uploaded files
```

#### Task Model (`task.py`)

**TaskStatus Enum:**
```python
class TaskStatus(str, Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"
```

**Task Dataclass:**
```python
@dataclass
class Task:
    task_id: str
    task_type: str
    status: TaskStatus
    created_at: datetime
    updated_at: datetime
    progress: int = 0              # 0-100
    message: str = ""
    result: Optional[Dict] = None
    error: Optional[str] = None
    metadata: Dict = field(default_factory=dict)
    progress_detail: Dict = field(default_factory=dict)
```

**TaskManager Implementation:**
- Uses **Singleton pattern** with thread-safe locking
- In-memory task storage (Dict) with optional cleanup of old tasks
- `cleanup_old_tasks(max_age_hours=24)` - Removes completed/failed tasks older than 24 hours

#### HistoryDatabase Vue Component

**Key Features:**

1. **扇形折叠布局 (Fan-folded collapsed state):**
   - Cards are stacked in a fan pattern when collapsed
   - Transform: `translate(${x}px, ${y}px) rotate(${r}deg) scale(${s})`
   - Center cards larger/smaller offset based on position

2. **网格展开布局 (Grid expanded state):**
   - 4 cards per row (`CARDS_PER_ROW = 4`)
   - Dynamic height calculation: `rows * CARD_HEIGHT + (rows - 1) * CARD_GAP`
   - 700ms cubic-bezier animation transition

3. **IntersectionObserver for auto-expand:**
   ```javascript
   threshold: [0.4, 0.6, 0.8],
   rootMargin: '0px 0px -150px 0px'
   ```
   - Debounced state changes to prevent flicker
   - Animation lock (`isAnimating`) prevents state conflicts

4. **Project Card Display:**
   - Simulation ID truncated: `SIM_{prefix.upper()}`
   - File type badges with Morandi color scheme
   - Status icons: `◇` Graph, `◈` Environment, `◆` Report
   - Progress states: `not-started`, `in-progress`, `completed`

5. **Modal for Project Details:**
   - Navigation buttons: Step1 (Graph), Step2 (Environment), Step4 (Report)
   - Note: "Step3 and Step5 require active simulation, not replayable"

### API Integration

**Frontend API (`frontend/src/api/simulation.js`):**
```javascript
export const getSimulationHistory = (limit = 20) => {
  return service.get('/api/simulation/history', { params: { limit } })
}
```

**Backend Endpoint (`backend/app/api/simulation.py`):**
- `GET /api/simulation/history` - Returns enriched simulation list with project details
- Includes: `simulation_id`, `project_id`, `project_name`, `simulation_requirement`, `status`, `entities_count`, `profiles_count`, `entity_types`, `total_rounds`, `current_round`, `report_id`

### Clever Solutions

1. **JSON-based persistence** - Avoids database setup, easy to debug
2. **Singleton TaskManager** - Thread-safe global access for async tasks
3. **File caching for extracted text** - Avoids re-parsing on subsequent operations
4. **Debounced IntersectionObserver** - Prevents rapid expand/collapse flickering
5. **Animation state machine** - `isAnimating` lock with pending state queue

### Technical Debt & Concerns

1. **No database transactions** - File-based storage vulnerable to concurrent access
2. **In-memory TaskManager** - Tasks lost on server restart
3. **No pagination** - `list_projects()` loads all projects into memory
4. **Project ID collision possible** - UUID hex slice (12 chars) has collision risk
5. **No project versioning** - Overwrites existing project data
6. **Hardcoded cleanup age** - `max_age_hours=24` not configurable

### Error Handling

- Project operations wrapped in try/except with logging
- File existence checks before operations
- Graceful degradation when files missing
- No validation on project IDs or data integrity

---

## Feature 11: Platform-Specific Simulations

### Core Implementation Files

| File | Size | Purpose |
|------|------|---------|
| `backend/scripts/run_reddit_simulation.py` | 27KB | OASIS Reddit simulation runner |
| `backend/scripts/run_twitter_simulation.py` | 27KB | OASIS Twitter simulation runner |

### Architecture Overview

Both scripts are **standalone execution scripts** that use the OASIS (Open AI Simulator for Interactive Simulations) framework from `camel-ai`.

#### Common Architecture

Both scripts share identical structure:

1. **IPCHandler** - File-based inter-process communication
2. **SimulationRunner** - Main simulation orchestration
3. **Signal handlers** - Graceful shutdown on SIGTERM/SIGINT

#### IPC Mechanism

**Command/Response Pattern:**
```
simulation_dir/
  ipc_commands/     # JSON files with commands
    {command_id}.json
  ipc_responses/    # JSON files with responses
    {command_id}.json
  env_status.json   # Current environment status
```

**Command Types:**
```python
class CommandType:
    INTERVIEW = "interview"
    BATCH_INTERVIEW = "batch_interview"
    CLOSE_ENV = "close_env"
```

**IPCHandler Core Methods:**
- `poll_command()` - Reads oldest JSON from `ipc_commands/`
- `send_response()` - Writes JSON to `ipc_responses/`, deletes command file
- `update_status()` - Writes to `env_status.json`
- `process_commands()` - Main IPC loop

#### Reddit Simulation Runner

**Available Actions:**
```python
AVAILABLE_ACTIONS = [
    ActionType.LIKE_POST,
    ActionType.DISLIKE_POST,
    ActionType.CREATE_POST,
    ActionType.CREATE_COMMENT,
    ActionType.LIKE_COMMENT,
    ActionType.DISLIKE_COMMENT,
    ActionType.SEARCH_POSTS,
    ActionType.SEARCH_USER,
    ActionType.TREND,
    ActionType.REFRESH,
    ActionType.DO_NOTHING,
    ActionType.FOLLOW,
    ActionType.MUTE,
]
```

**Profile Format:** `reddit_profiles.json`

**Database:** `reddit_simulation.db` (SQLite)

#### Twitter Simulation Runner

**Available Actions:**
```python
AVAILABLE_ACTIONS = [
    ActionType.CREATE_POST,
    ActionType.LIKE_POST,
    ActionType.REPOST,
    ActionType.FOLLOW,
    ActionType.DO_NOTHING,
    ActionType.QUOTE_POST,
]
```

**Profile Format:** `twitter_profiles.csv`

**Database:** `twitter_simulation.db` (SQLite)

#### Agent Activation Logic

Both runners use identical activation logic:

```python
def _get_active_agents_for_round(self, env, current_hour, round_num):
    time_config = self.config.get("time_config", {})
    agent_configs = self.config.get("agent_configs", [])

    # Base range
    base_min = time_config.get("agents_per_hour_min", 5)
    base_max = time_config.get("agents_per_hour_max", 20)

    # Time-based multipliers
    if current_hour in peak_hours:
        multiplier = time_config.get("peak_activity_multiplier", 1.5)
    elif current_hour in off_peak_hours:
        multiplier = time_config.get("off_peak_activity_multiplier", 0.3)
    else:
        multiplier = 1.0

    target_count = int(random.uniform(base_min, base_max) * multiplier)

    # Filter candidates by active_hours and activity_level
    for cfg in agent_configs:
        if current_hour not in cfg.get("active_hours", list(range(8, 23))):
            continue
        if random.random() < cfg.get("activity_level", 0.5):
            candidates.append(agent_id)

    return random.sample(candidates, min(target_count, len(candidates)))
```

#### Interview Mechanism

**Single Interview:**
```python
async def handle_interview(self, command_id, agent_id, prompt):
    agent = self.agent_graph.get_agent(agent_id)
    interview_action = ManualAction(
        action_type=ActionType.INTERVIEW,
        action_args={"prompt": prompt}
    )
    await self.env.step({agent: interview_action})
    result = self._get_interview_result(agent_id)
    self.send_response(command_id, "completed", result=result)
```

**Batch Interview:**
```python
async def handle_batch_interview(self, command_id, interviews):
    actions = {}
    for interview in interviews:
        agent = self.agent_graph.get_agent(interview["agent_id"])
        actions[agent] = ManualAction(
            action_type=ActionType.INTERVIEW,
            action_args={"prompt": interview.get("prompt", "")}
        )
    await self.env.step(actions)
```

#### Logging Configuration

Both scripts configure identical logging:
```python
loggers_config = {
    "social.agent": "social.agent.log",
    "social.twitter": "social.twitter.log",  # or "social.reddit" for Reddit
    "social.rec": "social.rec.log",
    "oasis.env": "oasis.env.log",
    "table": "table.log",
}
```

**Custom Unicode Formatter:**
```python
class UnicodeFormatter(logging.Formatter):
    UNICODE_ESCAPE_PATTERN = re.compile(r'\\u([0-9a-fA-F]{4})')

    def format(self, record):
        result = super().format(record)
        # Converts \uXXXX to readable characters
        return self.UNICODE_ESCAPE_PATTERN.sub(replace_unicode, result)
```

**MaxTokens Warning Filter:**
```python
class MaxTokensWarningFilter(logging.Filter):
    def filter(self, record):
        if "max_tokens" in record.getMessage() and "Invalid or missing" in record.getMessage():
            return False
        return True
```

#### Signal Handling

Both scripts register identical signal handlers:
```python
def setup_signal_handlers():
    def signal_handler(signum, frame):
        global _cleanup_done
        if not _cleanup_done:
            _cleanup_done = True
            _shutdown_event.set()
        else:
            sys.exit(1)

    signal.signal(signal.SIGTERM, signal_handler)
    signal.signal(signal.SIGINT, signal_handler)
```

### Clever Solutions

1. **File-based IPC** - No network overhead, easy to debug
2. **Polling with timeout** - `asyncio.wait_for(_shutdown_event.wait(), timeout=0.5)` allows responsive shutdown
3. **SQLite for simulation data** - Lightweight, no server required
4. **Profile CSV/JSON flexibility** - Different platforms can use different formats
5. **Unicode escape sequence conversion** - Makes OASIS logs readable
6. **MaxTokens warning suppression** - Reduces log noise (intentional design choice)
7. **Environment stay-alive mode** - Allows post-simulation interaction via IPC

### Technical Debt & Concerns

1. **Code duplication** - Reddit and Twitter scripts are nearly identical
2. **Polling-based IPC** - Inefficient compared to async message queues
3. **No authentication on IPC** - Any process can send commands
4. **Profile format mismatch** - Reddit uses JSON, Twitter uses CSV (inconsistent)
5. **No simulation state persistence** - Lost on crash
6. **Hardcoded semaphore=30** - Max concurrent LLM requests not configurable per-platform
7. **No retry logic on LLM failures** - Simulation halts on API error
8. **Duplicate code blocks** - `setup_oasis_logging`, `UnicodeFormatter`, `MaxTokensWarningFilter`, `IPCHandler` all duplicated

### Error Handling

- **Initial event failures** - Caught and logged with `try/except`
- **Agent lookup failures** - Warning printed, simulation continues
- **Interview failures** - Error returned via IPC response
- **Database errors** - Caught and logged, returns partial results
- **LLM API errors** - Propagates up, simulation stops

---

## Feature 12: Data Input Processing

### Core Implementation Files

| File | Size | Purpose |
|------|------|---------|
| `backend/app/services/text_processor.py` | 2KB | High-level text processing interface |
| `backend/app/utils/file_parser.py` | 5KB | Multi-format file parsing |

### Architecture Overview

#### FileParser (`file_parser.py`)

**Supported Formats:**
```python
SUPPORTED_EXTENSIONS = {'.pdf', '.md', '.markdown', '.txt'}
```

**Encoding Detection Fallback Chain:**
```python
def _read_text_with_fallback(file_path: str) -> str:
    # 1. Try UTF-8
    # 2. charset_normalizer.from_bytes(data).best()
    # 3. chardet.detect(data)
    # 4. Fallback to UTF-8 with errors='replace'
```

**Text Extraction Methods:**

1. **PDF Extraction (PyMuPDF):**
   ```python
   @staticmethod
   def _extract_from_pdf(file_path: str) -> str:
       import fitz  # PyMuPDF
       text_parts = []
       with fitz.open(file_path) as doc:
           for page in doc:
               text = page.get_text()
               if text.strip():
                   text_parts.append(text)
       return "\n\n".join(text_parts)
   ```

2. **Markdown/TXT (with encoding detection):**
   ```python
   @staticmethod
   def _extract_from_md(file_path: str) -> str:
       return _read_text_with_fallback(file_path)
   ```

**Multi-file Extraction:**
```python
@classmethod
def extract_from_multiple(cls, file_paths: List[str]) -> str:
    all_texts = []
    for i, file_path in enumerate(file_paths, 1):
        try:
            text = cls.extract_text(file_path)
            filename = Path(file_path).name
            all_texts.append(f"=== 文档 {i}: {filename} ===\n{text}")
        except Exception as e:
            all_texts.append(f"=== 文档 {i}: {file_path} (提取失败: {str(e)}) ===")
    return "\n\n".join(all_texts)
```

#### Text Chunking (`split_text_into_chunks`)

**Smart Sentence Boundary Detection:**
```python
def split_text_into_chunks(text, chunk_size=500, overlap=50):
    # Tries to split at sentence boundaries
    separators = ['。', '！', '？', '.\n', '!\n', '?\n', '\n\n', '. ', '! ', '? ']

    for sep in separators:
        last_sep = text[start:end].rfind(sep)
        if last_sep != -1 and last_sep > chunk_size * 0.3:
            end = start + last_sep + len(sep)
            break
```

**Key behavior:**
- If text <= chunk_size, returns single chunk (if non-empty)
- Overlap slides the window: `start = end - overlap`
- Prefers sentence boundaries for natural chunking
- Falls back to hard boundary if no separator found

#### TextProcessor (`text_processor.py`)

**High-level Interface:**
```python
class TextProcessor:
    @staticmethod
    def extract_from_files(file_paths: List[str]) -> str:
        return FileParser.extract_from_multiple(file_paths)

    @staticmethod
    def split_text(text, chunk_size=500, overlap=50) -> List[str]:
        return split_text_into_chunks(text, chunk_size, overlap)

    @staticmethod
    def preprocess_text(text: str) -> str:
        # Normalize newlines
        text = text.replace('\r\n', '\n').replace('\r', '\n')
        # Remove 3+ consecutive newlines
        text = re.sub(r'\n{3,}', '\n\n', text)
        # Trim whitespace per line
        lines = [line.strip() for line in text.split('\n')]
        return '\n'.join(lines).strip()

    @staticmethod
    def get_text_stats(text: str) -> dict:
        return {
            "total_chars": len(text),
            "total_lines": text.count('\n') + 1,
            "total_words": len(text.split()),
        }
```

### Integration Points

**Used in Graph API (`backend/app/api/graph.py`):**
```python
from ..utils.file_parser import FileParser

# During file upload processing:
text = FileParser.extract_text(file_info["path"])
```

**Used in TextProcessor service:**
```python
from ..utils.file_parser import FileParser, split_text_into_chunks

# High-level extraction:
text = TextProcessor.extract_from_files(file_paths)
chunks = TextProcessor.split_text(text, chunk_size, overlap)
```

**Project Text Caching:**
```python
# graph.py - saves extracted text to project directory
ProjectManager.save_extracted_text(project.project_id, all_text)

# Later retrieval
text = ProjectManager.get_extracted_text(project_id)
```

### Clever Solutions

1. **Multi-level encoding fallback** - charset_normalizer -> chardet -> UTF-8 replace
2. **Smart chunk boundary detection** - Tries multiple sentence separators
3. **Error isolation per file** - `extract_from_multiple` continues on individual file failure
4. **Preserves file order** - Enumerates files as "文档 1", "文档 2", etc.
5. **Whitespace normalization** - `preprocess_text` handles cross-platform line endings

### Technical Debt & Concerns

1. **No file size limits** - Large PDFs could exhaust memory
2. **No streaming for large files** - Loads entire file into memory
3. **PyMuPDF dependency** - External library may not be installed
4. **Limited format support** - No DOCX, XLSX, HTML support
5. **No image/table extraction from PDF** - Only raw text
6. **Chunk overlap could create duplicate context** - Same sentence spans two chunks
7. **No language detection** - Assumes Chinese/English punctuation
8. **Markdown extraction doesn't parse structure** - Returns raw text, no header hierarchy

### Error Handling

- **FileNotFoundError** - Raised with path details
- **ValueError for unsupported formats** - Includes the extension
- **ImportError for PyMuPDF** - Clear installation instructions
- **UnicodeDecodeError** - Handled by fallback chain
- **Empty PDF pages** - Skipped (only non-empty pages added)

---

## Cross-Feature Integration

### Data Flow

```
User Upload
    │
    ▼
graph.py API (upload endpoint)
    │
    ▼
FileParser.extract_text() ──────► ProjectManager.save_file_to_project()
    │                                     │
    ▼                                     ▼
TextProcessor.preprocess_text()    saved to files/
    │
    ▼
Text chunks ──────────────────────► ProjectManager.save_extracted_text()
    │                                     │
    ▼                                     ▼
OntologyGenerator                 cached as extracted_text.txt
    │
    ▼
Knowledge Graph (Zep)
```

### Simulation History Flow

```
Simulation Run
    │
    ▼
run_reddit/twitter_simulation.py
    │
    ▼
SQLite: reddit/twitter_simulation.db
    │
    ▼
get_simulation_history() API
    │
    ▼
HistoryDatabase.vue (frontend)
```

### Platform Selection

```
User selects platform(s)
    │
    ├─────────────────────┐
    ▼                     ▼
Reddit simulation     Twitter simulation
    │                     │
    ▼                     ▼
generate_reddit_     generate_twitter_
agent_graph()        agent_graph()
    │                     │
    ▼                     ▼
reddit_profiles.json twitter_profiles.csv
```

---

## Security Considerations

1. **ProjectManager** - No authentication on project access
2. **File upload** - Limited to predefined extensions, no virus scanning
3. **IPC mechanism** - No authentication, any local process can send commands
4. **No input sanitization** - Project names, file paths could contain special characters
5. **Temporary file cleanup** - Old logs not automatically purged

---

## Performance Observations

1. **File parsing** - Synchronous, could block API requests
2. **Project listing** - Loads all project.json files on every call
3. **TaskManager** - In-memory only, no persistence
4. **Simulation DB** - SQLite, fine for single simulation scale
5. **No connection pooling** - Each request creates new file handles
6. **Large PDF memory** - PyMuPDF loads entire PDF

---

## Missing Features (Based on Code Analysis)

1. **No project export/import** - Can't transfer projects between instances
2. **No simulation pause/resume** - Must complete in one run
3. **No concurrent simulation limit** - Could overwhelm LLM API
4. **No simulation cloning** - Can't duplicate a simulation setup
5. **No platform-agnostic profiles** - Reddit/Twitter require different formats
6. **No bulk file upload** - One file at a time in some flows

---

## Summary

The fourth batch of features represents the **data input and output infrastructure** of MiroFish:

- **History/Project Management** provides user-facing persistence with a clever fan-fold UI, but relies on fragile JSON file storage
- **Platform-Specific Simulations** showcase sophisticated OASIS integration with IPC-based command waiting, but suffer from significant code duplication
- **Data Input Processing** offers solid multi-format parsing with smart encoding fallback, but has limited format support and no streaming for large files

These features are functional but show signs of **rapid prototyping**: file-based storage, duplicated code between platforms, and missing enterprise features like authentication and concurrency limits.
