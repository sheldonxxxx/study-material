# Tech Stack Analysis: get-shit-done

## Language and Runtime

| Property | Value |
|----------|-------|
| **Primary Language** | JavaScript |
| **Runtime** | Node.js |
| **Minimum Version** | Node.js >= 20.0.0 |
| **File Format** | CommonJS (.cjs) |
| **Type System** | None (plain JavaScript) |

## Project Structure

```
get-shit-done/
├── bin/                      # Root CLI entry point
│   └── install.js             # Primary CLI executable
├── get-shit-done/            # Nested CLI tool (gsd-tools)
│   └── bin/
│       ├── gsd-tools.cjs      # Main CLI for nested tool
│       └── lib/*.cjs         # Core library modules
├── hooks/                    # Claude Code hooks
│   └── dist/                 # Built hooks for distribution
├── commands/                  # Command definitions
├── agents/                    # Agent configurations
├── workflows/                 # Workflow templates
├── templates/                 # Project templates
├── tests/                     # Test suite (~50 test files)
├── scripts/                   # Build and utility scripts
└── package.json
```

## Key Dependencies

### Runtime Dependencies
None explicitly listed in `dependencies` - the package is self-contained with all logic in `.cjs` files.

### Development Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| **c8** | ^11.0.0 | Code coverage measurement using V8's native coverage |
| **esbuild** | ^0.24.0 | JavaScript bundler (used for hook builds) |

## Build System

### Scripts

| Script | Command | Purpose |
|--------|---------|---------|
| `build:hooks` | `node scripts/build-hooks.js` | Validates syntax and copies hooks to `hooks/dist/` |
| `prepublishOnly` | `npm run build:hooks` | Runs before npm publish |
| `test` | `node scripts/run-tests.cjs` | Runs test suite via Node's built-in test runner |
| `test:coverage` | `c8 ... node scripts/run-tests.cjs` | Runs tests with coverage enforcement (70% line threshold) |

### Build Process

The build system is minimal:
1. **Hook validation** - Uses `vm.compileFunction` to validate JavaScript syntax without executing
2. **Hook copying** - Copies 5 hook files from `hooks/` to `hooks/dist/`
3. **No bundling** - Hooks are pure Node.js, no esbuild bundling actually used for hooks (esbuild is listed but build-hooks.js uses `fs.copyFileSync`)

### Test Framework

- **Test Runner**: Node.js built-in `--test` flag
- **Test Files**: 50+ `.test.cjs` files in `tests/` directory
- **Coverage**: c8 with 70% line coverage requirement
- **Execution**: Custom `run-tests.cjs` script handles cross-platform test file discovery

## CI/CD Configuration

### GitHub Actions Workflows

#### 1. `test.yml` - Test Workflow

| Property | Value |
|----------|-------|
| **Trigger** | push to main, pull requests to main, manual dispatch |
| **Timeout** | 10 minutes |
| **Concurrency** | Cancels in-progress on new run |

**Matrix Strategy:**
- OS: ubuntu-latest, macos-latest, windows-latest
- Node.js: 22, 24 (ubuntu); 22 (macOS/Windows)

**Steps:**
1. Checkout (v6.0.2)
2. Setup Node.js with npm cache
3. `npm ci` - clean install
4. `npm run test:coverage` - coverage enforcement

#### 2. `security-scan.yml` - Security Workflow

| Property | Value |
|----------|-------|
| **Trigger** | Pull requests to main |
| **Timeout** | 5 minutes |

**Scans Performed:**
- Prompt injection scan (`scripts/prompt-injection-scan.sh`)
- Base64 obfuscation scan (`scripts/base64-scan.sh`)
- Secret scan (`scripts/secret-scan.sh`)
- Planning directory check (ensures `.planning/` runtime data not committed)

### Other GitHub Configurations
- `auto-label-issues.yml` - Auto-labels GitHub issues
- `FUNDING.yml` - GitHub Sponsors integration
- Issue templates: bug, feature request, docs

## Binaries and Entry Points

### Root CLI (`bin/install.js`)
- Entry point: `get-shit-done-cc`
- Installs Claude Code hooks and CLI tools
- Cross-platform (Windows, macOS, Linux)

### Nested CLI (`get-shit-done/bin/gsd-tools.cjs`)
- Core GSD tools CLI
- Contains ~15 library modules in `get-shit-done/bin/lib/`:
  - `commands.cjs`, `config.cjs`, `core.cjs`, `frontmatter.cjs`
  - `init.cjs`, `milestone.cjs`, `model-profiles.cjs`, `phase.cjs`
  - `profile-output.cjs`, `profile-pipeline.cjs`, `roadmap.cjs`
  - `security.cjs`, `state.cjs`, `template.cjs`, `uat.cjs`
  - `verify.cjs`, `workstream.cjs`

## Security Features

1. **Prompt injection scanning** - Custom shell scripts scan for injection patterns
2. **Base64 obfuscation detection** - Scans for encoded content
3. **Secret scanning** - Prevents credential commits
4. **Planning directory guard** - Ensures runtime `.planning/` data excluded from commits
5. **Syntax validation** - Hooks validated before distribution

## Notable Observations

1. **No TypeScript** despite complex architecture
2. **No monorepo tooling** despite nested `get-shit-done/` directory (it's a CLI tool bundled with the main package)
3. **Pure CommonJS** - No ESM, no bundling for main code
4. **Minimal dependencies** - Only 2 dev dependencies for a sophisticated system
5. **Comprehensive test coverage** - 50+ test files with 70% coverage requirement
6. **Multi-platform CI** - Tests on Ubuntu, macOS, Windows
7. **Node.js version matrix** - Supports Node 22 and 24 in CI
