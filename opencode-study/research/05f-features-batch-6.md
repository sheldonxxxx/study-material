# Features 16-18: Git Integration, Cloud Infrastructure (SST), Container Images

## Feature 16: Git Integration

### Core Implementation

**Primary File:** `packages/opencode/src/git/index.ts` (309 lines)

The Git integration is built as an Effect service layer providing a clean abstraction over git operations. It uses `effect/unstable/process` for spawning git child processes.

#### Key Architecture Decisions

**1. Effect-based Service Layer**

```typescript
export class Service extends ServiceMap.Service<Service, Interface>()("@opencode/Git") {}

export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    const spawner = yield* ChildProcessSpawner.ChildProcessSpawner
    // ...
  }),
)
```

**2. Global Git Configuration**

The module pre-configures git with safe defaults applied to every command:

```typescript
const cfg = [
  "--no-optional-locks",
  "-c", "core.autocrlf=false",
  "-c", "core.fsmonitor=false",
  "-c", "core.longpaths=true",
  "-c", "core.symlinks=true",
  "-c", "core.quotepath=false",
] as const
```

This ensures consistent behavior across platforms.

**3. Interface Methods**

| Method | Purpose |
|--------|---------|
| `run(args, opts)` | Generic git command runner |
| `branch(cwd)` | Get current branch name |
| `prefix(cwd)` | Get path prefix within repo |
| `defaultBranch(cwd)` | Detect default branch (main/master or remote HEAD) |
| `hasHead(cwd)` | Check if HEAD exists |
| `mergeBase(cwd, base, head)` | Find common ancestor |
| `show(cwd, ref, file, prefix)` | Show file at specific ref |
| `status(cwd)` | Get porcelain status |
| `diff(cwd, ref)` | Get changed files vs ref |
| `stats(cwd, ref)` | Get addition/deletion stats |

**4. Branch Detection Logic**

```typescript
const defaultBranch = Effect.fn("Git.defaultBranch")(function* (cwd: string) {
  const remote = yield* primary(cwd)  // origin > first remote > upstream
  if (remote) {
    const head = yield* run(["symbolic-ref", `refs/remotes/${remote}/HEAD`], { cwd })
    if (head.exitCode === 0) {
      const ref = out(head).replace(/^refs\/remotes\//, "")
      const name = ref.startsWith(`${remote}/`) ? ref.slice(`${remote}/`.length) : ""
      if (name) return { name, ref }
    }
  }
  // Fallback to local branches: configured > main > master
})
```

### Worktree Management

**Primary File:** `packages/opencode/src/worktree/index.ts` (600 lines)

The worktree system extends native git worktrees with project sandbox isolation.

#### Key Features

**1. Worktree Lifecycle**

- `makeWorktreeInfo()` - Generate unique name/branch/directory
- `create()` - Setup + boot in one operation
- `remove()` - Clean removal with branch cleanup
- `reset()` - Hard reset + clean + submodule update + restart scripts

**2. Slug-based Name Generation**

```typescript
function slugify(input: string) {
  return input
    .trim()
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, "-")
    .replace(/^-+/, "")
    .replace(/-+$/, "")
}
```

Names are auto-generated using `@opencode-ai/util/slug` with `opencode/` prefix.

**3. Platform-aware Path Handling**

```typescript
const canonical = Effect.fnUntraced(function* (input: string) {
  const abs = pathSvc.resolve(input)
  const real = yield* fsys.realPath(abs).pipe(Effect.catch(() => Effect.succeed(abs)))
  const normalized = pathSvc.normalize(real)
  return process.platform === "win32" ? normalized.toLowerCase() : normalized
})
```

**4. Worktree Removal Resilience**

The `remove()` function handles multiple failure scenarios:

```typescript
const removed = yield* git(["worktree", "remove", "--force", entry.path], { cwd: Instance.worktree })
if (removed.code !== 0) {
  // Check if actually removed (stale entries)
  const stale = yield* locateWorktree(parseWorktreeList(next.text), directory)
  if (stale?.path) {
    throw new RemoveFailedError({ message: removed.stderr || removed.text || "Failed to remove git worktree" })
  }
}
// Clean up branch
const branch = entry.branch?.replace(/^refs\/heads\//, "")
if (branch) {
  yield* git(["branch", "-D", branch], { cwd: Instance.worktree })
}
```

### VCS Service Layer

**File:** `packages/opencode/src/project/vcs.ts` (237 lines)

Provides higher-level VCS operations on top of the Git service:

- `init()` - Initialize VCS state
- `branch()` - Get current branch
- `defaultBranch()` - Get default branch
- `diff(mode)` - Get file diffs (working tree or branch comparison)

#### Event Subscription

```typescript
yield* bus.subscribe(FileWatcher.Event.Updated).pipe(
  Stream.filter((evt) => evt.properties.file.endsWith("HEAD")),
  Stream.runForEach((_evt) =>
    Effect.gen(function* () {
      const next = yield* Effect.promise(() => get())
      if (next !== value.current) {
        log.info("branch changed", { from: value.current, to: next })
        value.current = next
        yield* bus.publish(Event.BranchUpdated, { branch: next })
      }
    }),
  ),
  Effect.forkScoped,
)
```

Watches for HEAD file changes to detect branch switches.

### Usage Points

| File | Usage |
|------|-------|
| `src/storage/storage.ts` | Migration: finding project roots via git |
| `src/project/vcs.ts` | VCS abstraction layer |
| `src/worktree/index.ts` | Git operations for worktree mgmt |
| `src/file/watcher.ts` | Ignoring .git directory |
| `src/cli/cmd/github.ts` | Git remote URL parsing, fork handling |
| `src/cli/cmd/pr.ts` | PR checkout with gh CLI |

### GitHub Integration (CLI)

**File:** `packages/opencode/src/cli/cmd/github.ts` (1647 lines)

A comprehensive GitHub App integration that:

1. Installs GitHub App via browser flow
2. Creates `.github/workflows/opencode.yml` workflow
3. Handles PR comments, issue comments, reviews
4. Manages fork remote setup
5. Extracts opencode session links from PR bodies

### PR Command (CLI)

**File:** `packages/opencode/src/cli/cmd/pr.ts` (127 lines)

Simple PR checkout that:
1. Fetches and checks out PR branch
2. Handles cross-repository forks
3. Extracts and imports opencode sessions from PR body
4. Launches opencode with session

---

## Feature 17: Cloud Infrastructure (SST)

### Overview

OpenCode uses **SST (Serverless Stack)** with **Cloudflare** as the provider for its cloud infrastructure. The architecture separates concerns into multiple infrastructure modules.

### Configuration

**File:** `sst.config.ts`

```typescript
export default $config({
  app(input) {
    return {
      name: "opencode",
      removal: input?.stage === "production" ? "retain" : "remove",
      protect: ["production"].includes(input?.stage),
      home: "cloudflare",
      providers: {
        stripe: { apiKey: process.env.STRIPE_SECRET_KEY! },
        planetscale: "0.4.1",
      },
    }
  },
  async run() {
    await import("./infra/app.js")
    await import("./infra/console.js")
    await import("./infra/enterprise.js")
  },
})
```

### Infrastructure Modules

#### 1. Domain Configuration

**File:** `infra/stage.ts`

```typescript
export const domain = (() => {
  if ($app.stage === "production") return "opencode.ai"
  if ($app.stage === "dev") return "dev.opencode.ai"
  return `${$app.stage}.dev.opencode.ai`
})()

export const shortDomain = (() => {
  if ($app.stage === "production") return "opncd.ai"
  if ($app.stage === "dev") return "dev.opncd.ai"
  return `${$app.stage}.dev.opncd.ai`
})()
```

#### 2. Database (PlanetScale)

**File:** `infra/app.ts` (lines 1-51)

```typescript
const cluster = planetscale.getDatabaseOutput({
  name: "opencode",
  organization: "anomalyco",
})

const branch =
  $app.stage === "production"
    ? planetscale.getBranchOutput({ name: "production", ... })
    : new planetscale.Branch("DatabaseBranch", {
        parentBranch: "production",  // Dev branches from production
      })
```

**Key pattern:** Development stages branch from production with `parentBranch: "production"`.

#### 3. Authentication Worker

```typescript
const authStorage = new sst.cloudflare.Kv("AuthStorage")
export const auth = new sst.cloudflare.Worker("AuthApi", {
  domain: `auth.${domain}`,
  handler: "packages/console/function/src/auth.ts",
  link: [database, authStorage, GITHUB_CLIENT_ID_CONSOLE, ...],
})
```

#### 4. Stripe Integration

**Products:**
- "ZenLite" (OpenCode Go) - $10/month, 50% first month coupon
- "ZenBlack" (OpenCode Black) - $200/$100/$20 tiered pricing

**Webhook endpoint** captures 20+ Stripe events including subscription lifecycle, payment events, and customer changes.

#### 5. Console Application

```typescript
new sst.cloudflare.x.SolidStart("Console", {
  domain,
  path: "packages/console/app",
  link: [bucket, bucketNew, database, auth, stripe, ...],
  environment: {
    VITE_AUTH_URL: auth.url.apply((url) => url!),
    VITE_STRIPE_PUBLISHABLE_KEY: STRIPE_PUBLISHABLE_KEY.value,
  },
  transform: {
    server: {
      placement: { region: "aws:us-east-1" },  // US-East for SolidStart
      transform: {
        worker: {
          tailConsumers: [{ service: logProcessor.nodes.worker.scriptName }],
        },
      },
    },
  },
})
```

#### 6. API Worker

**File:** `infra/console.ts`

```typescript
export const api = new sst.cloudflare.Worker("Api", {
  domain: `api.${domain}`,
  handler: "packages/function/src/api.ts",
  link: [bucket, GITHUB_APP_ID, ADMIN_SECRET, ...],
  transform: {
    worker: (args) => {
      args.logpush = true
      args.bindings = $resolve(args.bindings).apply((bindings) => [
        ...bindings,
        { name: "SYNC_SERVER", type: "durable_object_namespace", className: "SyncServer" },
      ])
    },
  },
})
```

**Key features:**
- `SyncServer` Durable Object for real-time sync
- Migration support with `oldTag`/`newTag` for backwards compatibility

#### 7. Web Properties

```typescript
// Documentation site
new sst.cloudflare.x.Astro("Web", {
  domain: "docs." + domain,
  path: "packages/web",
  environment: { SST_STAGE: $app.stage, VITE_API_URL: api.url.apply((url) => url!) },
})

// Web application
new sst.cloudflare.StaticSite("WebApp", {
  domain: "app." + domain,
  path: "packages/app",
  build: { command: "bun turbo build", output: "./dist" },
})
```

### Secrets Management

The infrastructure uses SST Secrets for sensitive configuration:

```typescript
const GITHUB_APP_ID = new sst.Secret("GITHUB_APP_ID")
const GITHUB_APP_PRIVATE_KEY = new sst.Secret("GITHUB_APP_PRIVATE_KEY")
const STRIPE_SECRET_KEY = new sst.Secret("STRIPE_SECRET_KEY")
// ... 30 ZEN model secrets (ZEN_MODELS1 through ZEN_MODELS30)
```

### Multi-Stage Support

| Stage | Domain | Database Branch |
|-------|--------|----------------|
| production | opencode.ai | production |
| dev | dev.opencode.ai | dev |
| others | {stage}.dev.opencode.ai | {stage} (from production) |

### Notable Clever Solutions

1. **Dual KV Storage:** Separate buckets for migration (`bucket`, `bucketNew`)
2. **Database Branching:** Dev stages branch from production for realistic testing
3. **Durable Objects for Sync:** SyncServer as DO namespace instead of external service
4. **Migration Tags:** Versioned migrations with oldTag/newTag for zero-downtime deploys

---

## Feature 18: Container Images

### Overview

CI-optimized container images at `packages/containers/` for faster GitHub Actions builds. These images pre-bake large, slow-to-install dependencies.

### Image Stack

```
base (Ubuntu 24.04 + common tools)
  |
  +-- bun-node (base + Node.js 24 + Bun 1.3.11)
        |
        +-- rust (bun-node + Rust stable)
              |
              +-- tauri-linux (rust + Tauri Linux build deps)
  +-- publish (bun-node + Docker CLI + AUR tooling)
```

### Dockerfiles

#### Base Image

**File:** `packages/containers/base/Dockerfile`

```dockerfile
ARG REGISTRY=ghcr.io/anomalyco
FROM ${REGISTRY}/build/bun-node:24.04

ARG RUST_TOOLCHAIN=stable

ENV CARGO_HOME=/opt/cargo
ENV RUSTUP_HOME=/opt/rustup
ENV PATH=/opt/cargo/bin:/opt/bun/bin:...

RUN set -euo pipefail; \
  curl -fsSL https://sh.rustup.rs | sh -s -- -y --profile minimal --default-toolchain "${RUST_TOOLCHAIN}"
```

**Key features:**
- Inherits from their own `bun-node` image
- Installs Rust with minimal profile
- Sets up proper PATH for cargo

#### Bun-Node Image

**File:** `packages/containers/bun-node/Dockerfile`

```dockerfile
ARG REGISTRY=ghcr.io/anomalyco
FROM ${REGISTRY}/build/base:24.04

SHELL ["/bin/bash", "-lc"]

ARG NODE_VERSION=24.4.0
ARG BUN_VERSION=1.3.11

ENV BUN_INSTALL=/opt/bun
ENV PATH=/opt/bun/bin:...

RUN set -euo pipefail; \
  arch=$(uname -m); \
  node_arch=x64; \
  if [ "$arch" = "aarch64" ]; then node_arch=arm64; fi; \
  curl -fsSL "https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-${node_arch}.tar.xz" \
  | tar -xJf - -C /usr/local --strip-components=1; \
  corepack enable

RUN set -euo pipefail; \
  curl -fsSL https://bun.sh/install | bash -s -- "bun-v${BUN_VERSION}"
```

**Key features:**
- Dual architecture support (x64 + arm64)
- Explicit version pinning (Node 24.4.0, Bun 1.3.11)
- corepack enabled for package manager consistency
- Uses bash login shell for bun installation

#### Rust Image

**File:** `packages/containers/rust/Dockerfile`

```dockerfile
ARG REGISTRY=ghcr.io/anomalyco
FROM ${REGISTRY}/build/rust:24.04

ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
  && apt-get install -y --no-install-recommends \
    libappindicator3-dev \
    libwebkit2gtk-4.1-dev \
    librsvg2-dev \
    patchelf \
  && rm -rf /var/lib/apt/lists/*
```

**Purpose:** Tauri application build dependencies.

#### Tauri-Linux Image

**File:** `packages/containers/tauri-linux/Dockerfile`

```dockerfile
FROM ubuntu:24.04

ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
  && apt-get install -y --no-install-recommends \
    build-essential \
    ca-certificates \
    curl \
    git \
    jq \
    openssh-client \
    pkg-config \
    python3 \
    unzip \
    xz-utils \
    zip
```

**Purpose:** Standalone build environment for general Linux builds.

### Build Script

**File:** `packages/containers/script/build.ts`

```typescript
const images = ["base", "bun-node", "rust", "tauri-linux", "publish"]

for (const name of images) {
  const image = `${reg}/build/${name}:${tag}`
  const file = `packages/containers/${name}/Dockerfile`

  if (name === "base") {
    // No registry argument for base
    if (push) {
      await $`docker buildx build --platform ${platform} -f ${file} -t ${image} --push .`
    }
  }
  if (name === "bun-node") {
    // Pass REGISTRY and BUN_VERSION args
  }
  // For other images, pass REGISTRY only
}
```

**Key features:**
- Multi-arch builds (`linux/amd64,linux/arm64`) via Buildx
- `REGISTRY` and `TAG` environment variables
- `--push` flag for publishing
- Docker buildx setup for multi-platform

### Registry

All images published to: `ghcr.io/anomalyco/build/{image}:{tag}`

Example: `ghcr.io/anomalyco/build/bun-node:24.04`

### Usage in CI

```yaml
jobs:
  build-cli:
    runs-on: ubuntu-latest
    container:
      image: ghcr.io/anomalyco/build/bun-node:24.04
```

### Technical Debt / Observations

1. **No caching strategy documented:** Build scripts don't explicitly use Docker layer caching
2. **Image versioning tied to date:** Tag `24.04` suggests monthly release cadence
3. **Limited image documentation:** No explanation of when to use which image
4. **Publish image mystery:** `publish` directory exists but Dockerfile not shown (likely multi-stage)

---

## Summary

| Feature | Architecture | Key Tech | Lines |
|---------|-------------|----------|-------|
| Git Integration | Effect Service Layer | effect, ChildProcess | ~1100 combined |
| Cloud Infrastructure | SST + Cloudflare | SST, PlanetScale, Stripe | ~300 combined |
| Container Images | Multi-stage Docker | Docker Buildx | ~100 combined |

All three features show mature, production-ready implementations with careful attention to cross-platform compatibility, error handling, and security (secrets management, OIDC tokens). |
