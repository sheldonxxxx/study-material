# memory-lancedb-pro — Learning Reference

## What This Is
Analysis of the memory-lancedb-pro codebase (v1.1.0-beta.10) — an OpenClaw plugin for enhanced LanceDB-backed memory with hybrid retrieval, smart extraction, and multi-scope isolation.

## Key Takeaways
- **Hybrid retrieval** (vector + BM25 + RRF fusion + cross-encoder reranking) is the core differentiation — 11-stage pipeline with configurable providers
- **jiti-based TypeScript execution** (no build step) is a clever productivity pattern — enables direct TS execution while maintaining type checking via `tsc`
- **Multi-scope ACL** with reflection scope auto-grant provides flexible isolation without complex hierarchies
- **Weibull decay** with tier-specific curves (Core β=0.8, Working β=1.0, Peripheral β=1.3) models memory degradation realistically
- **Two-stage deduplication** (cosine pre-filter → LLM decision) dramatically reduces expensive LLM calls

## Should Trust / Should Not Trust
- **Trust**: Architecture patterns, retrieval pipeline design, multi-provider adapter pattern, security measures (OAuth PKCE, SQL escaping, prompt injection prevention)
- **Do Not Trust**: README claims about code quality being "well-structured" — no tsconfig.json, 126 `any` escape hatches, no ESLint/Prettier, files up to 2,078 lines

## At a Glance
- **Language:** TypeScript (Node.js 22, ESM-only)
- **Architecture:** Layered (Plugin → Tool → Retrieval → Storage) with factory pattern
- **Key Libraries:** LanceDB, OpenAI SDK, TypeBox, Commander, jiti
- **Notable Patterns:** Strategy (retrieval modes), Adapter (rerankers), Factory (object creation), Proxy/Cache (LRU embedding cache), Observer (access tracking)
- **Stars / Activity:** Very high (376 commits in March 2026, 50+ contributors)
- **License:** Unknown (examine repo for details)

## Study Structure
```
memory-lancedb-pro-study/
├── learning/           # Synthesized learning documents
│   ├── README.md       # This file
│   ├── 01-project-overview.md
│   ├── 02-architecture.md
│   ├── 03-tech-stack.md
│   ├── 04-features-deep-dive.md
│   ├── 05-code-quality.md
│   ├── 06-ci-cd.md
│   ├── 07-documentation.md
│   ├── 08-security.md
│   ├── 09-dependencies.md
│   ├── 10-community.md
│   ├── 11-patterns.md
│   ├── 12-lessons-learned.md
│   └── 13-my-action-items.md
└── research/          # Preserved evidence from parallel subagent analysis
    ├── 01-topology.md
    ├── 02-tech-stack.md
    ├── 03-community.md
    ├── 04-features-index.md
    ├── 05a-features-batch-1.md
    ├── 05b-features-batch-2.md
    ├── 05c-features-batch-3.md
    ├── 05d-features-batch-4.md
    ├── 06-architecture.md
    ├── 07-code-quality.md
    └── 08-security-perf.md
```

## Most Valuable Patterns to Steal
1. **Two-stage dedup** — fast cosine pre-filter before expensive LLM dedup decision
2. **Multi-key round-robin** with rate-limit failover for embedding APIs
3. **Debounced batch write-back** — logarithmic frequency saturation for high-frequency access tracking
4. **Weibull decay** with tier-specific beta curves for realistic memory degradation
5. **CJK-aware thresholds** — language-specific length thresholds for noise filtering
6. **Cross-process file locking** with `proper-lockfile` for atomic LanceDB operations
7. **Injection prevention** via pattern matching on reflection content
8. **Adaptive intent routing** — rule-based (no LLM) for retrieval strategy selection

## Priority Action Items (from 13-my-action-items.md)
1. **P0:** Add tsconfig.json strict mode, ESLint + Prettier + pre-commit hooks
2. **P0:** Create constants.ts with documented magic numbers (decay weights, cache sizes, etc.)
3. **P1:** Implement serialized update queue with cross-process file locking
4. **P1:** Error class hierarchy instead of generic Error everywhere
5. **P2:** LRU embedding cache with TTL (256 entries, 30min TTL)
6. **P2:** Admission control with weighted multi-factor scoring
7. **P3:** Weibull decay lifecycle with tier-specific curves
