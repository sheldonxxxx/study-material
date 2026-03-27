# Code Quality Assessment

**Repo:** get-shit-done
**Assessment Date:** 2026-03-26
**Files Analyzed:** 17 core library modules (`.cjs`), 43 test files, CONTRIBUTING.md, package.json

---

## 1. Testing Approach

### Framework
The project uses **Node.js built-in test runner** (`node:test`) and assertion library (`node:assert/strict`). No external test frameworks (Jest, Mocha, Chai) are used.

```javascript
const { describe, it, test, beforeEach, afterEach } = require('node:test');
const assert = require('node:assert/strict');
```

**CONTRIBUTING.md** explicitly prohibits external test frameworks:
> "All tests use Node.js built-in test runner (`node:test`) and assertion library (`node:assert`). Do not use Jest, Mocha, Chai, or any external test framework."

### Coverage
- **Coverage tool:** `c8` (v11) for code coverage reporting
- **Coverage target:** 70% line coverage minimum
- **Command:** `npm run test:coverage`
- **Covered files:** Only `get-shit-done/bin/lib/*.cjs` (core library), explicitly excluding `tests/**`

### Test Structure

Tests follow a consistent pattern using `describe`/`test` blocks with `beforeEach`/`afterEach` hooks for setup/cleanup:

```javascript
describe('featureName', () => {
  let tmpDir;

  beforeEach(() => {
    tmpDir = createTempProject();
  });

  afterEach(() => {
    cleanup(tmpDir);
  });

  test('handles normal case', () => {
    assert.strictEqual(result, expected);
  });
});
```

**CONTRIBUTING.md** mandates hooks over `try/finally` for cleanup:
> "Always use `beforeEach`/`afterEach` for setup and cleanup. Do not use `try/finally` blocks for test cleanup -- they are verbose, error-prone, and can mask test failures."

### Test Helpers
Centralized test utilities in `tests/helpers.cjs`:
| Helper | Creates |
|--------|---------|
| `createTempProject(prefix?)` | tmpDir with `.planning/phases/` |
| `createTempGitProject(prefix?)` | Same + git init + initial commit |
| `createTempDir(prefix?)` | Bare temp directory |
| `cleanup(tmpDir)` | Removes directory recursively |
| `runGsdTools(args, cwd, env?)` | Executes gsd-tools.cjs CLI |

### Test File Count
43 test files covering all major modules. Key test files:
- `core.test.cjs` - 1,568 lines covering config loading, path utilities, phase search, git worktree handling
- `security.test.cjs` - Input validation, path traversal, prompt injection detection
- `state.test.cjs` - STATE.md operations
- `phase.test.cjs`, `roadmap.test.cjs`, `milestone.test.cjs` - Phase lifecycle
- `config.test.cjs` - Configuration CRUD
- `verify.test.cjs`, `uat.test.cjs` - Verification and UAT flows

### Node.js Compatibility
Tests must pass on **Node 22** (LTS) and **Node 24** (Current). No deprecated APIs used.

---

## 2. Type Safety

### No TypeScript
This is a **CommonJS JavaScript project** with no type system. All files use `.cjs` extension and `require()` syntax.

**Evidence:**
- No `*.ts` files, no `tsconfig.json`
- No `.d.ts` type definition files
- All imports use `require()` not ESM `import`
- package.json has no TypeScript-related dependencies

### Runtime Validation Instead
The project compensates for lack of static typing with extensive **runtime validation**:

**security.cjs** provides:
- `validatePath()` - Path traversal prevention with type checking
- `validatePhaseNumber()` - Phase number format validation
- `validateFieldName()` - Regex injection prevention
- `safeJsonParse()` - Safe JSON parsing with size limits
- `scanForInjection()` - Prompt injection detection
- `sanitizeForPrompt()` / `sanitizeForDisplay()` - Text sanitization

**config.cjs** validates config keys:
```javascript
function isValidConfigKey(keyPath) {
  if (VALID_CONFIG_KEYS.has(keyPath)) return true;
  if (/^agent_skills\.[a-zA-Z0-9_-]+$/.test(keyPath)) return true;
  return false;
}
```

### JSDoc Absent
No JSDoc comments for type hints. Function documentation is via regular comments.

---

## 3. Code Style and Formatting

### No ESLint or Prettier
The project has **no ESLint configuration** and **no Prettier configuration**. There is no `.eslintrc*` or `eslint.config.*` file anywhere in the repository.

### Conventional Commits
The project uses **conventional commits** for git messages:
- `feat:` - New features
- `fix:` - Bug fixes
- `docs:` - Documentation
- `refactor:` - Code refactoring
- `test:` - Test additions/changes
- `ci:` - CI configuration changes

CONTRIBUTING.md enforces: "No drive-by formatting -- don't reformat code unrelated to your change."

### File Naming
- Core library: `*.cjs` (CommonJS)
- Test files: `*.test.cjs`
- Scripts: `*.js`
- Hook scripts: `gsd-*.js` (prefix required)

### Markdown Normalization
The codebase enforces markdown style rules via `normalizeMd()` in core.cjs:
- **MD022** - Blank lines around headings
- **MD031** - Blank lines around fenced code blocks
- **MD032** - Blank lines around lists
- **MD012** - No 3+ consecutive blank lines
- **MD047** - Files end with single newline

Applied automatically at write points to keep generated `.planning/` files IDE-friendly.

### Gitignore Patterns
Standard ignore patterns in `.gitignore`:
- `node_modules/`
- `.planning/` (internal planning documents)
- `coverage/`
- `.claude/` (local test installs)
- `hooks/dist/` (build artifacts)

---

## 4. Error Handling Patterns

### Error Function Pattern
The codebase uses a centralized `error()` function in core.cjs that writes to stderr and exits with code 1:

```javascript
function error(message) {
  fs.writeSync(2, 'Error: ' + message + '\n');
  process.exit(1);
}
```

This is used across all library modules for fatal errors.

### Result Object Pattern
For non-fatal operations, functions return result objects rather than throwing:

```javascript
function validatePath(filePath, baseDir, opts = {}) {
  if (!filePath || typeof filePath !== 'string') {
    return { safe: false, resolved: '', error: 'Empty or invalid file path' };
  }
  // ...
  return { safe: true, resolved: resolvedPath };
}
```

### Safe Read Pattern
File reads use try/catch with null returns:

```javascript
function safeReadFile(filePath) {
  try {
    return fs.readFileSync(filePath, 'utf-8');
  } catch {
    return null;
  }
}
```

### Lock-based Concurrency Control
`withPlanningLock()` in core.cjs provides file-based locking for `.planning/` writes to prevent concurrent worktree corruption:

```javascript
function withPlanningLock(cwd, fn) {
  const lockPath = path.join(planningDir(cwd), '.lock');
  // Atomic lock acquisition, stale lock recovery after 30s
  // ...
}
```

### Empty Catch Blocks
The codebase commonly uses empty catch blocks for non-critical operations:

```javascript
try {
  fs.writeFileSync(configPath, JSON.stringify(parsed, null, 2), 'utf-8');
} catch { /* intentionally empty */ }
```

CONTRIBUTING.md does not explicitly address catch block style.

### Graceful Degradation
- `loadConfig()` catches JSON parse errors and returns defaults
- `safeJsonParse()` never throws, always returns `{ok, value?, error?}`
- `reapStaleTempFiles()` catches all errors non-critically

---

## 5. Configuration Management

### Three-Level Config Merge
`config.cjs`'s `buildNewProjectConfig()` merges configuration from three sources (in order of increasing priority):

1. **Hardcoded defaults** - Every key that `loadConfig()` resolves
2. **User-level defaults** - `~/.gsd/defaults.json`
3. **userChoices** - Settings from `/gsd:new-project`

```javascript
return {
  ...hardcoded,
  ...userDefaults,
  ...choices,
  // Nested objects deep-merged separately
  git: { ...hardcoded.git, ...(userDefaults.git || {}), ...(choices.git || {}) },
  workflow: { ... },
};
```

### Config Key Validation
Config keys are validated against `VALID_CONFIG_KEYS` Set with pattern matching for dynamic keys like `agent_skills.<agent-type>`.

### Config Auto-Detection
`loadConfig()` auto-detects:
- API key availability (brave search, firecrawl, exa)
- Gitignored `.planning/` directories (sets `commit_docs: false`)
- Sub-repos in monorepo workspaces (scans for child `.git/` directories)

### State File Pattern
STATE.md uses synchronized **YAML frontmatter** for machine-readable fields, with body text for human readability:

```javascript
function syncStateFrontmatter(content, cwd) {
  const existingFm = extractFrontmatter(content);
  const body = stripFrontmatter(content);
  const derivedFm = buildStateFrontmatter(body, cwd);
  return `---\n${yamlStr}\n---\n\n${body}`;
}
```

---

## 6. Quality Tooling

### Test Runner
- `npm test` - Runs all `.test.cjs` files via `node --test`
- `npm run test:coverage` - Coverage check at 70% line threshold

### Secret Scanning
Scripts directory contains security scanning tools:
- `secret-scan.sh` - Scans for exposed secrets
- `prompt-injection-scan.sh` - Scans for prompt injection patterns
- `base64-scan.sh` - Scans for base64-encoded content

### Build Scripts
- `scripts/build-hooks.js` - Compiles hook scripts (run at `prepublishOnly`)
- `scripts/run-tests.cjs` - Cross-platform test runner

### No CI Configuration in Repo
No `.github/workflows/` or `.gitlab-ci.yml` found. CI configuration is likely managed externally or in a separate repository.

### Package Scripts Summary

| Script | Purpose |
|--------|---------|
| `npm test` | Run all tests with `node --test` |
| `npm run test:coverage` | Coverage check at 70% threshold |
| `npm run build:hooks` | Build hook scripts |
| `prepublishOnly` | Auto-runs build before npm publish |

---

## Summary

| Dimension | Assessment |
|-----------|------------|
| **Testing** | Comprehensive Node.js built-in test framework, 43 test files, 70% coverage target, centralized helpers |
| **Type Safety** | None (CommonJS JS). Compensates with extensive runtime validation in security.cjs |
| **Code Style** | No automated linter/formatter. Relies on conventional commits and manual discipline |
| **Error Handling** | Centralized `error()` for fatal, result objects for recoverable, empty catch for non-critical |
| **Config Management** | Three-level merge with auto-detection, YAML frontmatter sync for STATE.md |
| **Tooling** | Test runner, coverage (c8), secret/injection scanners, no ESLint/Prettier |

### Strengths
- Comprehensive test coverage with dedicated helpers
- Strong security posture (path validation, prompt injection detection)
- Well-documented patterns in CONTRIBUTING.md
- Clean separation of concerns in lib modules

### Weaknesses
- No static type checking
- No automated code style enforcement
- Heavy reliance on runtime validation instead of static analysis
- Empty catch blocks can hide errors silently
