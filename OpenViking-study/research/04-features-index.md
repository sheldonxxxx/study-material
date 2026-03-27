# OpenViking Feature Index

**Project:** OpenViking - Context Database for AI Agents
**Repository:** /Users/sheldon/Documents/claw/reference/OpenViking
**Generated:** 2026-03-27

## Overview

OpenViking is a context database designed for AI Agents, implementing a "filesystem paradigm" to unify memories, resources, and skills management. The system uses tiered context loading (L0/L1/L2), directory recursive retrieval, and automatic session management.

---

## Core Features (Priority 1)

### 1. Virtual Filesystem (viking:// Protocol)
**Description:** Core abstraction layer implementing a virtual filesystem paradigm for unified context management. All memories, resources, and skills are mapped to virtual directories under the `viking://` protocol with unique URIs.

**Key Files/Locations:**
- `openviking/storage/viking_fs.py` - Main virtual filesystem implementation (70KB)
- `openviking/storage/vikingdb_manager.py` - Database manager
- `openviking/storage/local_fs.py` - Local filesystem adapter
- `docs/en/concepts/04-viking-uri.md` - URI specification

---

### 2. Tiered Context Loading (L0/L1/L2)
**Description:** Automatic context processing into three hierarchy levels: L0 (abstract/summary ~100 tokens), L1 (overview ~2KB for planning), L2 (full content loaded on demand). Reduces token consumption significantly.

**Key Files/Locations:**
- `openviking/storage/compressor.py` - Content compression engine (30KB)
- `openviking/storage/compressor_v2.py` - V2 compression
- `openviking/storage/memory_deduplicator.py` - Deduplication before storage
- `docs/en/concepts/03-context-layers.md` - Layer specification

---

### 3. Directory Recursive Retrieval
**Description:** Hierarchical retrieval strategy combining intent analysis, vector-based initial positioning, and recursive directory exploration. Improves retrieval accuracy by understanding full context of information location.

**Key Files/Locations:**
- `openviking/retrieve/hierarchical_retriever.py` - Main retriever (24KB)
- `openviking/retrieve/intent_analyzer.py` - Query intent analysis
- `openviking/retrieve/retrieval_stats.py` - Retrieval statistics
- `docs/en/concepts/07-retrieval.md` - Retrieval mechanism details

---

### 4. Vector Database Integration
**Description:** Pluggable vector database backend supporting multiple embedding providers (Volcengine, OpenAI, Jina, Voyage, MiniMax, VikingDB, Gemini). Enables semantic search with configurable dimensions.

**Key Files/Locations:**
- `openviking/storage/vectordb/` - Vector DB implementations
- `openviking/storage/vectordb_adapters/` - Provider adapters
- `openviking/storage/viking_vector_index_backend.py` - Vector indexing
- `docs/en/concepts/05-storage.md` - Storage architecture

---

### 5. Session Management & Memory Self-Iteration
**Description:** Automatic session management that compresses conversations, extracts long-term memory, and iteratively improves agent context. Updates both user preferences and agent experience.

**Key Files/Locations:**
- `openviking/session/session.py` - Session handling (37KB)
- `openviking/session/memory_extractor.py` - Memory extraction (62KB)
- `openviking/session/memory_archiver.py` - Memory archiving
- `openviking/session/memory/` - Memory submodules
- `docs/en/concepts/08-session.md` - Session specification

---

### 6. VikingBot AI Agent Framework
**Description:** AI agent framework built on OpenViking, providing channels (CLI, etc.), agent implementations, and OpenViking mount integration for context access.

**Key Files/Locations:**
- `bot/vikingbot/` - Main bot package
- `bot/vikingbot/agent/` - Agent implementations
- `bot/vikingbot/channels/` - Communication channels
- `bot/vikingbot/openviking_mount/` - Context mounting
- `bot/vikingbot/providers/` - Model providers
- `examples/quick_start.py` - Quick start example

---

### 7. Multi-Provider Model Support
**Description:** Unified support for VLM and embedding models across providers: Volcengine (Doubao), OpenAI, LiteLLM (Anthropic/Claude, DeepSeek, Gemini, Qwen, vLLM, Ollama, etc.).

**Key Files/Locations:**
- `openviking/server/models.py` - Model configurations
- `openviking/client/async_client.py` - Async client
- `openviking/client/sync_client.py` - Sync client
- `examples/openclaw-plugin/` - Plugin examples

---

## Secondary Features (Priority 2)

### 8. Rust CLI (ov_cli)
**Description:** High-performance Rust-based command-line client with TUI for interactive use. Alternative to Python client with additional terminal features.

**Key Files/Locations:**
- `crates/ov_cli/` - Rust CLI source
- `crates/ov_cli/src/main.rs` - Main entry (35KB)
- `crates/ov_cli/src/client.rs` - Client implementation (26KB)
- `crates/ov_cli/src/commands/` - CLI commands
- `crates/ov_cli/src/tui/` - Terminal UI

---

### 9. HTTP Server & REST API
**Description:** FastAPI-based HTTP server providing persistent context API for AI agents. Includes routers for resources, sessions, retrieval, and admin functions.

**Key Files/Locations:**
- `openviking/server/app.py` - FastAPI application
- `openviking/server/bootstrap.py` - Server bootstrap
- `openviking/server/routers/` - API route definitions
- `openviking/server/config.py` - Server configuration
- `openviking/server/auth.py` - Authentication

---

### 10. AGFS (Abstract Google File System) - C++ Backend
**Description:** C++ implementation of a distributed filesystem-like abstraction for high-performance context storage and retrieval. Built with cross-platform support.

**Key Files/Locations:**
- `openviking_cli/` - C++/Python hybrid package
- `openviking_cli/store/` - Storage implementations
- `openviking_cli/index/` - Index implementations
- `openviking_cli/common/` - Shared utilities
- `openviking/agfs_manager.py` - Python AGFS manager

---

### 11. OpenClaw/OpenCode Plugin Integration
**Description:** Context plugins for external agent frameworks (OpenClaw, OpenCode) demonstrating OpenViking integration as a drop-in memory backend.

**Key Files/Locations:**
- `examples/openclaw-plugin/` - OpenClaw plugin implementation
- `examples/opencode-memory-plugin/` - OpenCode memory plugin
- `examples/openclaw-plugin/README.md` - Integration details

---

### 12. Observability & Telemetry
**Description:** Retrieval trajectory visualization, telemetry collection, and statistics tracking for debugging and optimizing context management.

**Key Files/Locations:**
- `openviking/telemetry/` - Telemetry module
- `openviking/server/telemetry.py` - Server telemetry
- `openviking/retrieve/retrieval_stats.py` - Retrieval statistics
- `openviking/eval/` - Evaluation utilities

---

## Directory Structure Summary

```
openviking/                 # Python SDK
  storage/                  # VikingFS, vector DB, compression
  retrieve/                 # Hierarchical retrieval
  session/                  # Session & memory management
  server/                   # HTTP API server
  client/                   # Python client libraries
  telemetry/                # Observability
  eval/                     # Evaluation tools
  pyagfs/                   # Python AGFS bindings
  resource/                 # Resource management
  misc/                     # Miscellaneous utilities

bot/vikingbot/              # VikingBot agent framework
  agent/                    # Agent implementations
  channels/                 # Communication channels
  providers/                # Model providers
  openviking_mount/         # Context mounting

crates/ov_cli/              # Rust CLI tool
  src/commands/             # CLI command implementations
  src/tui/                  # Terminal UI

openviking_cli/             # C++ AGFS implementation
  store/                    # Storage backends
  index/                    # Index implementations

examples/                   # Usage examples
  openclaw-plugin/          # OpenClaw integration
  opencode-memory-plugin/   # OpenCode integration
  basic-usage/              # Basic usage examples

tests/                      # Test suites
docs/en/concepts/           # Architecture concepts
```

---

## Feature Priority Matrix

| Feature | Core/Secondary | Complexity | Files | Status |
|---------|---------------|------------|-------|--------|
| Virtual Filesystem (viking://) | Core | High | 5+ | Stable |
| Tiered Context Loading | Core | High | 4+ | Stable |
| Directory Recursive Retrieval | Core | High | 3+ | Stable |
| Vector DB Integration | Core | Medium | 4+ | Stable |
| Session Management | Core | High | 5+ | Stable |
| VikingBot Framework | Core | High | 10+ | Stable |
| Multi-Provider Support | Core | Medium | 4+ | Stable |
| Rust CLI | Secondary | Medium | 8+ | Stable |
| HTTP Server/API | Secondary | Medium | 6+ | Stable |
| AGFS C++ Backend | Secondary | High | 10+ | Stable |
| Plugin Integration | Secondary | Low | 4+ | Stable |
| Observability | Secondary | Low | 4+ | Stable |

---

## Requirement Coverage

From README documented challenges and solutions:

| Challenge | Solution | Feature |
|-----------|----------|---------|
| Fragmented Context | Filesystem Paradigm | Feature #1 |
| Surging Context Demand | Tiered Context Loading | Feature #2 |
| Poor Retrieval Effectiveness | Directory Recursive Retrieval | Feature #3 |
| Unobservable Context | Retrieval Trajectory Visualization | Feature #12 |
| Limited Memory Iteration | Automatic Session Management | Feature #5 |
| Multi-model Support | Provider Abstraction | Feature #7 |
| Agent Development | VikingBot Framework | Feature #6 |

---

*Generated from README.md analysis and directory structure mapping*
