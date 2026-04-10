# MiroFish Project Overview

## Project Type

**Full-stack Web Application** - AI-powered swarm intelligence simulation engine for predictive analytics

## What MiroFish Does

MiroFish is a next-generation AI prediction engine that uses multi-agent technology to construct parallel digital worlds for simulating and predicting outcomes. Users upload seed material (reports, stories) and describe prediction requirements in natural language.

The system orchestrates a 5-step workflow:

1. **Graph Build** - Construct knowledge graphs from uploaded documents using GraphRAG
2. **Environment Setup** - Configure simulation parameters and OASIS profile
3. **Simulation** - Execute multi-agent simulations in parallel (Twitter/Reddit platforms)
4. **Report Generation** - Generate analytical reports from simulation results
5. **Interaction** - Query and interact with the simulated agent memory via Zep

## Target Users

- Researchers studying predictive analytics
- Organizations requiring outcome simulation before decision-making
- Analysts working with social media trend prediction

## Architecture

### Backend (Flask + Python 3.11+)
- **Location**: `backend/`
- **Entry**: `backend/run.py` (port 5001)
- **Key Services**:
  - `simulation_manager.py` - Orchestrates simulation lifecycle
  - `simulation_runner.py` - Executes agent simulations (1764 lines)
  - `graph_builder.py` - Builds GraphRAG knowledge graphs
  - `zep_graph_memory_updater.py` - Manages agent memory via Zep service
  - `llm_client.py` - Interfaces with LLM providers (OpenAI-compatible)

### Frontend (Vue 3 + Vite)
- **Location**: `frontend/`
- **Entry**: `frontend/src/main.js` (port 3000)
- **Key Components**:
  - `Step1GraphBuild.vue` through `Step5Interaction.vue` - Workflow steps
  - `GraphPanel.vue` - D3-based knowledge graph visualization
  - `SimulationRunView.vue` - Real-time simulation monitoring

### External Services
- **LLM Provider**: Alibaba DashScope (compatible with OpenAI SDK)
- **Memory Service**: Zep (vector memory and entity tracking)

## Tech Stack Summary

| Layer | Technology |
|-------|------------|
| Backend Framework | Flask |
| Python Package Manager | uv |
| Frontend Framework | Vue 3 |
| Build Tool | Vite |
| HTTP Client (FE) | Axios |
| Visualization | D3.js |
| Knowledge Graph | Zep |
| Containerization | Docker + docker-compose |
| CI/CD | GitHub Actions |

## Key Features

- **Multi-Platform Simulation**: Supports Twitter and Reddit agent behaviors
- **Real-time Progress Tracking**: Server-Sent Events for simulation updates
- **Knowledge Graph Visualization**: Interactive D3 force-directed graphs
- **Parallel Agent Execution**: Thread-based simulation runners
- **Structured Logging**: Per-module loggers with rotation

## Repository Location

```
/Users/sheldon/Documents/claw/reference/MiroFish
```
