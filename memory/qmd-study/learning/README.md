# QMD — Learning Reference

## What This Is
Analysis of the QMD codebase — an on-device hybrid search engine for markdown knowledge bases — for informing development of similar projects in the local AI search space.

## Key Takeaways
- **Hybrid search works**: BM25 + vector + RRF fusion + LLM reranking is a proven combination for semantic search over local documents
- **Local LLMs are viable**: node-llama-cpp with GGUF models enables full-featured search without API dependencies
- **MCP as first-class integration**: QMD treats AI agent integration (via Model Context Protocol) as a primary use case, not an afterthought
- **Smart chunking matters**: Markdown-aware break-point scoring (rather than naive token splitting) significantly improves retrieval quality
- **Monolith has tradeoffs**: The 4379-line store.ts enables fast iteration but creates maintenance challenges

## Should Trust / Should Not Trust
- **Trust**: Search pipeline architecture, chunking algorithm, RRF fusion formula, MCP integration design
- **Do Not Trust**: README simplicity claims — the codebase is significantly more complex than docs suggest
- **Do Not Trust**: Code quality scores — the 5.9/10 assessment is fair but the project ships working software

## At a Glance
- **Language**: TypeScript (strict mode)
- **Architecture**: Layered monolith + MCP plugin
- **Key Libraries**: better-sqlite3, sqlite-vec, node-llama-cpp, @modelcontextprotocol/sdk
- **Notable Patterns**: Content-addressable storage (SHA256 docids), session-based LLM resource management, hierarchical context inheritance, dual-write config (YAML + SQLite)
- **Stars / Activity**: ~267 commits since Jan 2026, ~44 merged PRs, ~40 contributors
- **License**: MIT
- **Maintainer**: Tobi Lutke (Shopify cofounder)

## Repository Structure
```
qmd-study/
├── learning/          # Synthesized learning documents (start here)
│   ├── README.md       # This file
│   ├── 01-project-overview.md
│   ├── 02-architecture.md
│   ├── 03-tech-stack.md
│   ├── 04-features-deep-dive.md  # Most detailed feature analysis
│   ├── 05-code-quality.md
│   ├── 06-ci-cd.md
│   ├── 07-documentation.md
│   ├── 08-security.md
│   ├── 09-dependencies.md
│   ├── 10-community.md
│   ├── 11-patterns.md  # All design patterns catalogued
│   ├── 12-lessons-learned.md
│   └── 13-my-action-items.md     # Prioritized next steps
└── research/          # Raw research from subagents (preserve)
    ├── 01-topology.md
    ├── 02-tech-stack.md
    ├── 03-community.md
    ├── 04-features-index.md
    ├── 05a-features-batch-1.md   # Hybrid search, collection, context
    ├── 05b-features-batch-2.md   # MCP, SDK, chunking
    ├── 05c-features-batch-3.md   # Query expansion, output formats, retrieval
    ├── 05d-features-batch-4.md   # HTTP transport, embeddings, maintenance
    ├── 06-architecture.md
    ├── 07-code-quality.md
    └── 08-security-perf.md
```

## Recommended Reading Order
1. **Start**: `01-project-overview.md` (5 min) — understand what QMD is
2. **Then**: `04-features-deep-dive.md` (20 min) — deep dive into how features work
3. **Then**: `11-patterns.md` (10 min) — catalog of design patterns to emulate/avoid
4. **Then**: `12-lessons-learned.md` (10 min) — honest assessment of what to learn from
5. **Then**: `13-my-action-items.md` (5 min) — actionable next steps for your project
6. **Reference**: Remaining files as needed for specific domains

## Source Code References
- Main implementation: `src/store.ts` (4379 lines)
- LLM integration: `src/llm.ts` (1547 lines)
- MCP server: `src/mcp/server.ts` (808 lines)
- SDK facade: `src/index.ts` (529 lines)
- Collection config: `src/collections.ts` (501 lines)
