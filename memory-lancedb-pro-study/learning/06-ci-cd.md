# CI/CD

## GitHub Actions Workflows

The repository has four workflows in `.github/workflows/`:

### ci.yml

Main CI pipeline running on every push and pull request. Two jobs:

**version-sync job:**
- Checks that `package.json` version matches `openclaw.plugin.json` version
- Uses inline node script to parse and compare versions
- Fails with error if mismatch found

**cli-smoke job:**
- Runs on `ubuntu-latest`
- Uses `actions/setup-node@v4` with `node-version: 22`
- Caches npm via `cache: npm`
- Runs `npm ci` then `npm test`

### auto-assign.yml

Runs on issue and PR open events:
- Issues: auto-assigns to `AliceLJY`
- PRs (non-fork only): auto-assigns to `rwmjhb`
- Uses `pozil/auto-assign-issue@v1` action

### claude.yml

Triggers on:
- `issue_comment` (created)
- `pull_request_review_comment` (created)
- `issues` (opened, assigned)
- `pull_request_review` (submitted)

Conditional trigger when comment/body contains `@claude`. Runs `anthropics/claude-code-action@v1` with OAuth token. Reads CI results via `actions: read` permission.

### claude-code-review.yml

Runs on PR events: `opened`, `synchronize`, `ready_for_review`, `reopened`. Skips fork PRs. Uses `anthropics/claude-code-action@v1` with the `code-review@claude-code-plugins` plugin. Prompts: `/code-review:code-review ${{ github.repository }}/pull/${{ github.event.pull_request.number }}`

## Test Command

```bash
npm test
```

The test script chains 27 test files:
- `node test/embedder-error-hints.test.mjs`
- `node test/cjk-recursion-regression.test.mjs`
- `node test/migrate-legacy-schema.test.mjs`
- `node --test test/config-session-strategy-migration.test.mjs`
- `node --test test/scope-access-undefined.test.mjs`
- `node --test test/reflection-bypass-hook.test.mjs`
- `node --test test/smart-extractor-scope-filter.test.mjs`
- `node --test test/store-empty-scope-filter.test.mjs`
- `node --test test/recall-text-cleanup.test.mjs`
- `node test/update-consistency-lancedb.test.mjs`
- `node --test test/strip-envelope-metadata.test.mjs`
- `node test/cli-smoke.mjs`
- `node test/functional-e2e.mjs`
- `node test/retriever-rerank-regression.mjs`
- `node test/smart-memory-lifecycle.mjs`
- `node test/smart-extractor-branches.mjs`
- `node test/plugin-manifest-regression.mjs`
- `node --test test/sync-plugin-version.test.mjs`
- `node test/smart-metadata-v2.mjs`
- `node test/vector-search-cosine.test.mjs`
- `node test/context-support-e2e.mjs`
- `node test/temporal-facts.test.mjs`
- `node test/memory-update-supersede.test.mjs`
- `node test/memory-upgrader-diagnostics.test.mjs`
- `node --test test/llm-api-key-client.test.mjs`
- `node --test test/llm-oauth-client.test.mjs`
- `node --test test/cli-oauth-login.test.mjs`
- `node --test test/workflow-fork-guards.test.mjs`
- `node --test test/clawteam-scope.test.mjs`
- `node --test test/cross-process-lock.test.mjs`
- `node --test test/preference-slots.test.mjs`

A separate test command exists for openclaw host testing:
```bash
npm run test:openclaw-host
```

## Version Management

A `version` script syncs version between `openclaw.plugin.json` and `package.json`:
```bash
npm run version  # runs: node scripts/sync-plugin-version.mjs openclaw.plugin.json package.json && git add openclaw.plugin.json
```

CI enforces version consistency between `package.json` and `openclaw.plugin.json` as a required check.

## Release Process

- No automated release workflow detected (no publish to npm in CI)
- Releases appear to be manual with version bumps via the `npm run version` script
- Beta track (v1.1.0-beta.x) and stable track (v1.0.x) maintained
- Version sync enforced by CI before merge

## Key Observations

- **No deployment automation** -- no auto-publish to npm on tag/release
- **No test coverage reporting** -- no codecov or similar integration
- **No staging/production environments** -- plugin is distributed via npm
- **Version sync is a gate** -- mismatched versions will fail CI
- **No CODEOWNERS file** -- reviewer assignment relies on auto-assign workflow pointing to specific individuals
