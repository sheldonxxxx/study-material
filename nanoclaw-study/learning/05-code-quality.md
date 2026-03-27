# NanoClaw Code Quality

## Type System

NanoClaw uses **TypeScript** throughout with strict type checking enabled via `typescript-eslint`.

### TypeScript Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

### TypeScript-ESLint Rules

```javascript
// eslint.config.js
'@typescript-eslint/no-unused-vars': [
  'error',
  {
    args: 'all',
    argsIgnorePattern: '^_',
    caughtErrors: 'all',
    caughtErrorsIgnorePattern: '^_',
    destructuredArrayIgnorePattern: '^_',
    varsIgnorePattern: '^_',
    ignoreRestSiblings: true,
  },
],
'@typescript-eslint/no-explicit-any': 'warn',
```

**Evidence:** Unused variables, unused catch parameters, and `any` types are enforced or warned against. The `_` prefix convention allows intentional unused variables.

### Zod for Runtime Validation

Zod (v4.3.6) validates external data at runtime:

```typescript
// Example pattern in codebase - schema validation for structured data
import { z } from 'zod';

export const AdditionalMountSchema = z.object({
  hostPath: z.string(),
  containerPath: z.string().optional(),
  readonly: z.boolean().optional(),
});
```

### Type Export Pattern

Types are centralized in `src/types.ts` and exported for use across modules:

```typescript
// src/types.ts
export interface Channel {
  name: string;
  connect(): Promise<void>;
  sendMessage(jid: string, text: string): Promise<void>;
  isConnected(): boolean;
  ownsJid(jid: string): boolean;
  disconnect(): Promise<void>;
  setTyping?(jid: string, isTyping: boolean): Promise<void>;
  syncGroups?(force: boolean): Promise<void>;
}

export interface RegisteredGroup {
  name: string;
  folder: string;
  trigger: string;
  added_at: string;
  containerConfig?: ContainerConfig;
  requiresTrigger?: boolean;
  isMain?: true;
}
```

## Testing

NanoClaw uses **Vitest** (v4.0.18) for unit testing with coverage reporting via `@vitest/coverage-v8`.

### Test Configuration

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    include: ['src/**/*.test.ts', 'setup/**/*.test.ts'],
  },
});
```

### Test File Examples

**Unit tests exist for core modules:**

| File | Coverage |
|------|----------|
| `src/ipc-auth.test.ts` | IPC authorization (main vs non-main group permissions) |
| `src/sender-allowlist.test.ts` | Sender allowlist loading and validation |
| `src/routing.test.ts` | Message routing and format validation |
| `src/group-queue.test.ts` | Queue concurrency and ordering |
| `src/container-runtime.test.ts` | Container spawn and mount handling |
| `src/db.test.ts` | Database operations and migrations |
| `setup/environment.test.ts` | Environment validation |
| `setup/platform.test.ts` | Platform detection |

### IPC Authorization Tests (Evidence)

```typescript
// src/ipc-auth.test.ts - line 71-127
describe('schedule_task authorization', () => {
  it('main group can schedule for another group', async () => {
    await processTaskIpc(
      { type: 'schedule_task', prompt: 'do something', ... },
      'whatsapp_main',
      true,  // isMain = true
      deps,
    );
    const allTasks = getAllTasks();
    expect(allTasks.length).toBe(1);
    expect(allTasks[0].group_folder).toBe('other-group');
  });

  it('non-main group cannot schedule for another group', async () => {
    await processTaskIpc(
      { type: 'schedule_task', prompt: 'unauthorized', ... },
      'other-group',
      false, // isMain = false
      deps,
    );
    const allTasks = getAllTasks();
    expect(allTasks.length).toBe(0); // Task rejected
  });
});
```

### Test Database Pattern

Tests use an in-memory SQLite database:

```typescript
// src/db.ts - line 162-166
/** @internal - for tests only. Creates a fresh in-memory database. */
export function _initTestDatabase(): void {
  db = new Database(':memory:');
  createSchema(db);
}
```

## Linting and Formatting

### ESLint Configuration

```javascript
// eslint.config.js
export default [
  { ignores: ['node_modules/', 'dist/', 'container/', 'groups/'] },
  { files: ['src/**/*.{js,ts}'] },
  { languageOptions: { globals: globals.node } },
  pluginJs.configs.recommended,
  ...tseslint.configs.recommended,
  {
    plugins: { 'no-catch-all': noCatchAll },
    rules: {
      'preserve-caught-error': ['error', { requireCatchParameter: true }],
      'no-catch-all/no-catch-all': 'warn',
    },
  },
];
```

**Key rules:**
- `preserve-caught-error`: Requires catch parameter to be used (not ignored)
- `no-catch-all/no-catch-all`: Warns against catching `catch (e)` without specific types
- `@typescript-eslint/no-unused-vars`: Error on unused variables
- `@typescript-eslint/no-explicit-any`: Warning on `any` type usage

### Prettier Configuration

```json
// .prettierrc
{}
```

Uses Prettier with default settings for consistent formatting.

### npm Scripts

```json
// package.json
{
  "scripts": {
    "build": "tsc",
    "typecheck": "tsc --noEmit",
    "lint": "eslint src/",
    "lint:fix": "eslint src/ --fix",
    "format": "prettier --write \"src/**/*.ts\"",
    "format:check": "prettier --check \"src/**/*.ts\"",
    "test": "vitest run",
    "test:watch": "vitest"
  }
}
```

### Husky Pre-commit Hook

```json
// package.json
"prepare": "husky"
```

Husky runs lint and format checks before commits.

## Code Patterns

### Module Organization

- ESM modules throughout (`"type": "module"` in package.json)
- Explicit `.js` extensions in imports (`import { x } from './y.js'`)
- Barrel exports via `index.ts` files

### Error Handling

The codebase prefers specific error handling over catch-all:

```typescript
// src/sender-allowlist.ts - line 39-47
try {
  raw = fs.readFileSync(filePath, 'utf-8');
} catch (err: unknown) {
  if ((err as NodeJS.ErrnoException).code === 'ENOENT') return DEFAULT_CONFIG;
  logger.warn({ err, path: filePath }, 'sender-allowlist: cannot read config');
  return DEFAULT_CONFIG;
}
```

### Logging

Uses Pino logger with structured logging:

```typescript
// src/mount-security.ts - line 17-20
const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: { target: 'pino-pretty', options: { colorize: true } },
});
```

Log levels: debug, info, warn, error with contextual data via `logger.info({ key: value }, 'message')`.

### Database Migrations

Schema migrations handled via try/catch on ALTER TABLE:

```typescript
// src/db.ts - line 87-94
try {
  database.exec(
    `ALTER TABLE scheduled_tasks ADD COLUMN context_mode TEXT DEFAULT 'isolated'`,
  );
} catch {
  /* column already exists */
}
```

### Null Safety

Explicit null checks and validation:

```typescript
// src/db.ts - line 582-588
const row = db.prepare('SELECT * FROM registered_groups WHERE jid = ?').get(jid);
if (!row) return undefined;
if (!isValidGroupFolder(row.folder)) {
  logger.warn({ jid: row.jid, folder: row.folder }, 'Skipping invalid folder');
  return undefined;
}
```

## Quality Metrics

- **Test files**: 10+ `.test.ts` files covering core functionality
- **Type coverage**: TypeScript strict mode, Zod validation for external data
- **Linting**: ESLint + TypeScript-ESLint with recommended configs
- **Format**: Prettier with pre-commit hooks via Husky
- **Build verification**: `tsc --noEmit` for type checking in CI
