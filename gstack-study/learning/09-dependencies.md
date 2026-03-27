# Dependencies

## Runtime Dependencies

### playwright ^1.58.2

**Purpose:** Headless Chromium browser automation via Chrome DevTools Protocol (CDP).

**Why Playwright:**
- Cross-browser support (Chromium, Firefox, WebKit)
- Robust CDP support for advanced browser control
- Built-in locator system (`getByRole`, `getByText`, etc.) that doesn't require DOM mutation
- Active maintenance and large community

**gstack usage:**
- `browse/src/server.ts` — Bun.serve HTTP server that dispatches commands to Chromium via Playwright CDP
- `browse/src/cli.ts` — CLI that communicates with the server over localhost HTTP
- Cookie decryption reads Chromium's SQLite cookie database directly (Bun has native SQLite)

**Alternatives considered:**
- Puppeteer — less mature locator system, less active on CDP
- Selenium — heavy, slow, not designed for headless-first
- native CDP directly — too low-level, would require implementing browser lifecycle management

### diff ^7.0.0

**Purpose:** Text diffing for PR review output formatting.

**gstack usage:**
- `review/SKILL.md.tmpl` — for generating diff hunks in PR reviews

**Why this library:**
- Tree-shakeable ESM module
- No native dependencies
- Fast enough for diffing PR-sized changes

**Alternatives considered:**
- `diff-match-patch` — larger, more features than needed
- `jsdiff` — similar, diff is simpler

## Development Dependencies

### @anthropic-ai/sdk ^0.78.0

**Purpose:** LLM-as-judge scoring in Tier 3 evals.

**gstack usage:**
- `test/skill-llm-eval.test.ts` — calls Claude API directly to score generated SKILL.md docs

**Why devDependency:**
- Only used in test infrastructure, not at runtime
- Users installing gstack don't need this

## Dependency Philosophy

gstack maintains a **minimal dependency footprint**:

| Category | Count | Rationale |
|----------|-------|-----------|
| Runtime dependencies | 2 | Only browser automation (Playwright) and diffing are truly needed |
| Dev dependencies | 1 | SDK for LLM evals only |
| Total | 3 | Minimal attack surface, fast install |

## No Backend Dependencies

gstack has **no server-side components**:
- No database (state is file-based: `.gstack/browse.json`)
- No external API at runtime (only during evals via ANTHROPIC_API_KEY)
- No message queue, cache server, or other infrastructure

The CLI and server communicate over localhost HTTP within a single user's session.

## Telemetry Dependency

Telemetry (opt-in) sends data to Supabase. The schema is in `supabase/migrations/` and is publicly auditable. This is:
- Open source Firebase alternative
- Row-level security policies deny all direct access
- Publishable key is a public key (like Firebase)
- Flows through validated edge functions with allowlists

Users can verify exactly what's collected by reading the migration files.

## Platform-Specific Dependencies

### macOS
- **Keychain access** — for decrypting Chromium cookies. Uses native `security` CLI via `Bun.spawn()` with explicit argument arrays (no shell interpolation).

### Windows
- **Node.js** — required because Bun has a bug with Playwright's pipe transport (bun#4253). The browse server falls back to Node.js automatically.
- **Git Bash or WSL** — gstack works via these shells on Windows 11.

### Linux
- No additional dependencies beyond what the Docker image provides.

## Docker Image Dependencies

The CI image (`.github/docker/Dockerfile.ci`) pre-bakes:
- Bun runtime
- Node.js (for Windows fallback path)
- Playwright with Chromium browser
- All `package.json` dependencies

Cached at `/opt/node_modules_cache` and `/opt/playwright-browsers` for fast restoration.

## Dependency Update Strategy

- **Playwright** — tracked via `^1.58.2`. Update when new versions fix browser bugs or add features gstack needs.
- **diff** — rarely changes; update when major security issues arise.
- **@anthropic-ai/sdk** — only for evals, not critical path. Update when new Claude models have better eval performance.

No automated dependency updates (Dependabot or similar). Changes are evaluated manually during regular development.
