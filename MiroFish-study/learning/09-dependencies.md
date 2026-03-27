# MiroFish Dependencies

## Python Dependencies (Backend)

**File:** `backend/pyproject.toml`

### Core Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| flask | >=3.0.0 | Web framework |
| flask-cors | >=6.0.0 | CORS handling |
| openai | >=1.0.0 | LLM SDK |
| zep-cloud | 3.13.0 | Memory graph service |
| camel-oasis | 0.2.5 | Social media simulation |
| camel-ai | 0.2.78 | Multi-agent framework |
| PyMuPDF | >=1.24.0 | PDF/file processing |
| charset-normalizer | >=3.0.0 | Encoding detection |
| chardet | >=5.0.0 | Encoding detection |
| python-dotenv | >=1.0.0 | Environment variables |
| pydantic | >=2.0.0 | Data validation |

### Dev Dependencies

| Package | Version |
|---------|---------|
| pytest | >=8.0.0 |
| pytest-asyncio | >=0.23.0 |
| pipreqs | >=0.5.0 |

### Build System

- **Backend:** hatchling (PEP 517 build frontend)
- **Package:** `backend/pyproject.toml` with `tool.hatch.build.targets.wheel packages = ["app"]`

## Node.js Dependencies (Frontend)

**File:** `frontend/package.json`

### Production Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| vue | ^3.5.24 | UI framework |
| vue-router | ^4.6.3 | Client-side routing |
| axios | ^1.13.2 | HTTP client |
| d3 | ^7.9.0 | Data visualization |

### Dev Dependencies

| Package | Version |
|---------|---------|
| vite | ^7.2.4 | Build tool |
| @vitejs/plugin-vue | ^6.0.1 | Vue 3 Vite plugin |

## Root Dependencies

**File:** `package.json`

### Dev Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| concurrently | ^9.1.2 | Run frontend + backend simultaneously |

### Node Engine Requirement

- `node >=18.0.0`

## Package Manager Matrix

| Layer | Manager | Lock File |
|-------|---------|-----------|
| Root | npm | package-lock.json |
| Frontend | npm | frontend/package-lock.json |
| Backend | uv | backend/uv.lock |

## Key External Services

| Service | Purpose | Required |
|---------|---------|----------|
| Alibaba Bailian (LLM_API_KEY) | Primary LLM (qwen-plus) | Yes |
| Zep Cloud (ZEP_API_KEY) | Memory graph/agent memory | Yes |
| OpenAI-compatible API | Alternative LLM provider | No |

## Dependency Health Observations

1. **Pinned major versions** -- Most Python deps use `>=X.Y.Z` rather than exact pins
2. **Recent CAMEL versions** -- camel-oasis 0.2.5, camel-ai 0.2.78 (active development)
3. **Single version of Zep Cloud** -- 3.13.0 exact pin
4. **No pip fallback in production** -- uv is the primary package manager; requirements.txt exists but is not actively used
