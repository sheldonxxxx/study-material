# MiroFish Project Topology

## Project Type

**Full-stack Web Application** - AI-powered swarm intelligence simulation engine for predictive analytics

## Overview

MiroFish is a next-generation AI prediction engine that uses multi-agent technology to construct parallel digital worlds for simulating and predicting outcomes. Users upload seed material (reports, stories) and describe prediction requirements in natural language.

---

## Directory Structure

```
MiroFish/
├── backend/                    # Python Flask backend
│   ├── app/
│   │   ├── __init__.py       # Flask app factory (create_app)
│   │   ├── config.py         # Configuration management
│   │   ├── api/              # API route handlers
│   │   │   ├── graph.py      # GraphRAG endpoints
│   │   │   ├── report.py     # Report generation endpoints
│   │   │   └── simulation.py # Simulation control endpoints
│   │   ├── models/           # Data models
│   │   │   ├── project.py    # Project model
│   │   │   └── task.py       # Task model
│   │   ├── services/         # Business logic services
│   │   │   ├── graph_builder.py
│   │   │   ├── oasis_profile_generator.py
│   │   │   ├── ontology_generator.py
│   │   │   ├── report_agent.py
│   │   │   ├── simulation_config_generator.py
│   │   │   ├── simulation_ipc.py
│   │   │   ├── simulation_manager.py
│   │   │   ├── simulation_runner.py
│   │   │   ├── text_processor.py
│   │   │   ├── zep_entity_reader.py
│   │   │   ├── zep_graph_memory_updater.py
│   │   │   └── zep_tools.py
│   │   └── utils/            # Utility functions
│   │       ├── file_parser.py
│   │       ├── llm_client.py
│   │       ├── logger.py
│   │       ├── retry.py
│   │       └── zep_paging.py
│   ├── scripts/              # Backend scripts
│   ├── pyproject.toml        # Python dependencies (uv)
│   ├── requirements.txt
│   ├── uv.lock
│   └── run.py               # Backend entry point
│
├── frontend/                  # Vue.js 3 + Vite frontend
│   ├── public/              # Static public assets
│   ├── src/
│   │   ├── App.vue          # Root component
│   │   ├── main.js          # Frontend entry point
│   │   ├── api/             # Frontend API clients
│   │   │   ├── graph.js
│   │   │   ├── index.js
│   │   │   ├── report.js
│   │   │   └── simulation.js
│   │   ├── assets/          # Frontend assets
│   │   ├── components/      # Vue components
│   │   │   ├── GraphPanel.vue
│   │   │   ├── HistoryDatabase.vue
│   │   │   ├── Step1GraphBuild.vue
│   │   │   ├── Step2EnvSetup.vue
│   │   │   ├── Step3Simulation.vue
│   │   │   ├── Step4Report.vue
│   │   │   └── Step5Interaction.vue
│   │   ├── router/
│   │   │   └── index.js     # Vue Router configuration
│   │   ├── store/           # State management
│   │   └── views/           # Page-level components
│   │       ├── Home.vue
│   │       ├── InteractionView.vue
│   │       ├── MainView.vue
│   │       ├── Process.vue
│   │       ├── ReportView.vue
│   │       ├── SimulationRunView.vue
│   │       └── SimulationView.vue
│   ├── index.html
│   ├── package.json
│   ├── package-lock.json
│   └── vite.config.js
│
├── static/                   # Static assets (images)
├── .github/
│   └── workflows/           # CI/CD workflows
├── .env.example             # Environment template
├── .gitignore
├── .dockerignore
├── Dockerfile               # Multi-service Docker image
├── docker-compose.yml       # Docker orchestration
├── package.json            # Root workspace/package.json
├── package-lock.json
├── README.md
└── README-EN.md
```

---

## Entry Points

### Backend Entry Point
- **File**: `backend/run.py`
- **Function**: `main()`
- **Port**: 5001 (default)
- **Framework**: Flask (Python)

### Frontend Entry Point
- **File**: `frontend/src/main.js`
- **Mount**: `#app` (in `frontend/index.html`)
- **Framework**: Vue 3 + Vite
- **Port**: 3000 (default)

### Root Workspace
- **File**: `package.json` (root)
- **Scripts**: Orchestrates both frontend and backend
  - `npm run dev` - Start both services concurrently
  - `npm run backend` - Start backend only
  - `npm run frontend` - Start frontend only

---

## API Routes (Backend Blueprints)

| Blueprint | Prefix | Purpose |
|-----------|--------|---------|
| `graph_bp` | `/api/graph` | GraphRAG construction |
| `simulation_bp` | `/api/simulation` | Simulation control |
| `report_bp` | `/api/report` | Report generation |

### Health Check
- **Endpoint**: `GET /health`
- **Response**: `{"status": "ok", "service": "MiroFish Backend"}`

---

## Frontend Routes (Vue Router)

| Path | Component | Purpose |
|------|-----------|---------|
| `/` | Home | Landing page |
| `/process/:projectId` | Process | Main workflow process |
| `/simulation/:simulationId` | SimulationView | Simulation view |
| `/simulation/:simulationId/start` | SimulationRunView | Simulation execution |
| `/report/:reportId` | ReportView | Report viewing |
| `/interaction/:reportId` | InteractionView | Agent interaction |

---

## Key Technologies

### Backend
- **Framework**: Flask
- **Package Manager**: uv
- **Python Version**: 3.11+
- **Key Libraries**:
  - flask-cors (CORS handling)
  - LLM integration (OpenAI SDK format)
  - Zep (memory/knowledge graph)

### Frontend
- **Framework**: Vue 3
- **Build Tool**: Vite
- **Key Libraries**:
  - vue-router (routing)
  - axios (HTTP client)
  - d3 (data visualization)

### Infrastructure
- **Containerization**: Docker + docker-compose
- **CI/CD**: GitHub Actions

---

## Service Ports

| Service | Port | URL |
|---------|------|-----|
| Frontend | 3000 | http://localhost:3000 |
| Backend API | 5001 | http://localhost:5001 |

---

## Configuration

- **Backend config**: `backend/app/config.py`
- **Environment template**: `.env.example`
- **Required env vars**:
  - `LLM_API_KEY` - LLM API authentication
  - `LLM_BASE_URL` - LLM API endpoint
  - `LLM_MODEL_NAME` - Model identifier (e.g., qwen-plus)
  - `ZEP_API_KEY` - Zep memory service

---

## Docker

- **Single image**: Contains both frontend and backend
- **Start command**: `npm run dev` (runs concurrently)
- **Volumes**: `./backend/uploads:/app/backend/uploads`
