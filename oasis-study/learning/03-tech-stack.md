# Tech Stack

## Overview

OASIS (Open Agent Social Interaction Simulations) is a Python-based social media simulator that uses large language model agents to mimic real-world social media platforms at scale.

## Core Language & Runtime

| Component | Technology |
|-----------|------------|
| Language | Python 3.10 - 3.11 |
| Package Manager | Poetry |
| Build System | Poetry Core |

## Key Dependencies

### AI & LLM Framework
- **camel-ai 0.2.78** - CAMEL (Communicative Agents framework) provides the underlying multi-agent infrastructure
- **sentence-transformers 3.0.0** - Embedding models for content representation

### Data & Processing
- **pandas 2.2.2** - Data manipulation and analysis
- **numpy** (implicit via dependencies) - Numerical computing

### Graph & Network
- **igraph 0.11.6** - Graph data structures for social network modeling

### Visualization & Graphics
- **cairocffi 1.7.1** - Graphics library bindings for visualization
- **Pillow 10.3.0** - Image processing

### Platforms & Integrations
- **neo4j 5.23.0** - Graph database for social network storage
- **slack_sdk 3.31.0** - Slack integration
- **requests_oauthlib 2.0.0** - OAuth authentication

### Content Processing
- **unstructured 0.13.7** - Document parsing and content extraction

### Testing
- **pytest 8.2.0** - Unit testing framework
- **pytest-asyncio 0.23.6** - Async test support

### Documentation
- **Mintlify** - Documentation site generator (hosted at docs.oasis.camel-ai.org)

## Architecture

```
oasis/
├── __init__.py           # Package initialization, exports main API
├── clock/                # Time simulation components
├── environment/          # Simulation environment (PettingZoo-style)
├── social_agent/         # Agent definitions and behaviors
├── social_platform/      # Platform implementations (Reddit, Twitter, etc.)
├── testing/              # Test utilities
└── visualization/       # Data visualization tools
```

## Technical Decisions

1. **Python 3.10-3.11 constraint** - Balanced compatibility with dependencies (especially camel-ai)
2. **Poetry for packaging** - Modern Python dependency management with lock files
3. **PettingZoo-style interface** - Environment follows multi-agent RL conventions for interoperability
4. **Graph-based social networks** - igraph for efficient large-scale network operations
5. **Neo4j integration** - Native graph database suited for relationship-heavy social data

## Summary

OASIS builds on established AI/ML libraries (CAMEL, sentence-transformers) combined with graph databases (Neo4j) and social platform integrations to create a scalable social media simulation platform.
