# Dependencies

## Overview

OASIS has 19 direct dependencies managed via Poetry, with a locked dependency tree of 4222 lines in poetry.lock.

## Direct Dependencies (pyproject.toml)

```toml
[tool.poetry.dependencies]
python = ">=3.10.0,<3.12"

# Core dependencies
pandas = "2.2.2"
igraph = "0.11.6"
cairocffi = "1.7.1"
pillow = "10.3.0"

# AI/ML
camel-ai = "0.2.78"
sentence-transformers = "3.0.0"

# Testing
pytest = "8.2.0"
pytest-asyncio = "0.23.6"
pre-commit = "3.7.1"

# Integrations
neo4j = "5.23.0"
slack_sdk = "3.31.0"
requests_oauthlib = "2.0.0"

# Documentation & Validation
unstructured = "0.13.7"
prance = "23.6.21.0"
openapi-spec-validator = "0.7.1"
```

## Dependency Categories

### Core Simulation
- **pandas 2.2.2** - Data manipulation for simulation data
- **igraph 0.11.6** - Graph operations for social networks
- **cairocffi 1.7.1** - Graphics rendering for visualizations

### AI/ML Stack
- **camel-ai 0.2.78** - Multi-agent framework (CAMEL = Communicative Agents)
- **sentence-transformers 3.0.0** - Text embeddings for content similarity

### Platform Integrations
- **neo4j 5.23.0** - Graph database for persistent social graphs
- **slack_sdk 3.31.0** - Slack bot integration
- **requests_oauthlib 2.0.0** - OAuth for platform authentication

### Content Processing
- **unstructured 0.13.7** - Parsing diverse document formats

### Testing & DevOps
- **pytest 8.2.0** - Test framework
- **pytest-asyncio 0.23.6** - Async test support
- **pre-commit 3.7.1** - Git hooks

### Spec Validation
- **prance 23.6.21.0** - OpenAPI spec validation
- **openapi-spec-validator 0.7.1** - API specification checking

## Key Dependency: camel-ai

The camel-ai dependency (v0.2.78) is the most significant, providing:
- Multi-agent communication protocols
- LLM integration
- Tool calling infrastructure
- Memory systems

This is CAMEL-AI's flagship framework for building autonomous cooperative agent systems.

## Dependency Management

### Installation
```bash
poetry install
```

### Lock File Synchronization
When modifying `pyproject.toml`:
```bash
poetry lock  # Regenerate poetry.lock
```

### Dependency Pins
Most dependencies are loosely versioned (no exact pins), allowing minor updates while avoiding breaking changes.

## Python Version Constraint

Python 3.10-3.11 is explicitly required:
```toml
python = ">=3.10.0,<3.12"
```

This constraint likely exists due to compatibility requirements with camel-ai and sentence-transformers.

## External Services

| Service | Purpose | Configuration |
|---------|---------|---------------|
| OpenAI API | LLM provider | `OPENAI_API_KEY` env var |
| Neo4j | Graph database | Connection string |
| Slack | Team communication | OAuth tokens |

## Notable Absences

- No Docker/container runtime in dependencies (though `.container/` exists)
- No Redis or message queue (single-machine simulation)
- No cloud storage SDKs (local file-based data)

## Dependency Health

- **Active maintenance**: Last updated December 2025 (camel-ai bump to 0.2.78)
- **Security consideration**: Requires OpenAI API key for testing
- **Lock file integrity**: 4222-line poetry.lock suggests comprehensive dependency resolution
