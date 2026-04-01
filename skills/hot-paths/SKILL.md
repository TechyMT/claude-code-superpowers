---
name: hot-paths
description: |
  Teaches Claude Code's startup and runtime performance patterns: lazySchema() for deferred Zod parsing, feature() flags for build-time dead code elimination, memoize() for expensive repeated computations, parallel initialization with Promise.all(), and profileCheckpoint() for tracking load times. Use this whenever writing code that runs at startup, on every turn, or in a hot loop. Startup latency is Claude Code's most user-visible performance metric.
---

# Hot Paths

## The pattern

Claude Code has three hot paths: **startup** (module loading before first prompt), **per-turn** (context assembly before each API call), and **per-tool-call** (schema validation and permission checking for every tool use). Each has different performance constraints and different techniques.

For startup: use `lazySchema()` to defer Zod parsing, use `feature()` flags to eliminate dead code at build time, parallelize independent I/O with `Promise.all()`, and track timing with `profileCheckpoint()`.

For per-turn: memoize expensive computations (git status, file system state) that don't change mid-conversation. Clear memoized values only when the underlying state actually changes.

For per-tool-call: `lazySchema()` ensures schema objects are built at most once per process lifetime (on first access), not once per call.

## Why this matters

Claude Code starts a new process on every invocation. Users expect the prompt to appear in under a second. With ~512K lines of TypeScript across 1,884 files, naive module loading would be too slow. The techniques below are how the team keeps startup under a second on typical developer machines.

The per-turn hot path is also user-visible: every time the LLM needs to generate a response, Claude Code assembles a system prompt that includes git status, open file list, working directory, and recent file changes. This assembly runs before each API call. If it were slow, every turn would feel sluggish.

## How to apply it

**Startup:**
1. Wrap Zod schemas in `lazySchema(() => z.object({...}))` from `utils/lazySchema.ts`. The schema is built only when first accessed, not when the module loads.
2. Use `feature('FLAG_NAME')` from `bun:bundle` for conditional imports. Bun evaluates these at build time and tree-shakes unused code. An unused feature flag means the code is not included in the bundle.
3. Use `profileCheckpoint('label')` to measure time between initialization steps during development.
4. For independent async initialization (config reads, keychain prefetch, MCP connection), use `Promise.all()`.

**Per-turn:**
5. Use `memoize()` from lodash for functions that are expensive and called multiple times per conversation (git status, workspace file enumeration).
6. Clear memoized values via `cache.clear?.()` when the underlying state changes (e.g., after a file write or git commit).

**Per-tool-call:**
7. `lazySchema()` schemas are effectively memoized at module scope — no additional caching needed.
8. For permission rule matching, pre-compile patterns once and reuse them.

## In the source

```typescript
// Source: src/tools/FileReadTool/FileReadTool.ts (lazySchema for deferred init)
import { lazySchema } from '../../utils/lazySchema.js'

// Schema construction deferred until first tool invocation — not at module load
const inputSchema = lazySchema(() =>
  z.object({
    file_path: z.string(),
    limit: z.number().optional(),
    offset: z.number().optional(),
  })
)

// Source: src/main.tsx (feature flags for dead code elimination)
// Bun evaluates feature() at build time — unused modules not in bundle
const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js') as typeof import('./assistant/index.js')
  : null

const kairosGate = feature('KAIROS')
  ? require('./assistant/gate.js') as typeof import('./assistant/gate.js')
  : null

// Source: src/context.ts (memoize for per-conversation caching)
import { memoize } from 'lodash-es'

export const getGitStatus = memoize(async (): Promise<string | null> => {
  // This runs once per conversation, not once per turn
  // Five git commands parallelized — not sequential
  const [branch, mainBranch, status, log, userName] = await Promise.all([
    getBranch(),
    getDefaultBranch(),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'status', '--short']),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'log', '--oneline', '-n', '5']),
    execFileNoThrow(gitExe(), ['config', 'user.name']),
  ])
  // ...
})

// Cache is cleared when state changes (e.g., after git commit)
getGitStatus.cache.clear?.()

// Source: src/main.tsx (profileCheckpoint for startup profiling)
profileCheckpoint('imports_loaded')
await initializeConfig()
profileCheckpoint('config_loaded')
await Promise.all([
  prefetchKeychain(),
  connectMCPServers(),
])
profileCheckpoint('parallel_init_done')
```

The `--no-optional-locks` flag in git commands is a performance detail: by default git acquires a lock file before reading index state. `--no-optional-locks` skips that when the operation doesn't need the lock, saving ~20ms per command. On five parallel commands, this saves ~100ms.

## Apply it to your code

**Before** — schema built at module load, expensive init not parallelized:
```typescript
// Module load time: slow — Zod builds validators immediately
const inputSchema = z.object({
  file_path: z.string(),
  content: z.string(),
  encoding: z.enum(['utf-8', 'base64']).optional(),
})

// Sequential initialization
async function initWorkspace() {
  const config = await readConfig()      // 50ms
  const gitStatus = await getGitStatus() // 120ms
  const files = await listOpenFiles()    // 30ms
  // Total: ~200ms sequential
  return { config, gitStatus, files }
}
```

**After** — lazy schema, parallel init:
```typescript
// Schema deferred — no cost at module load time
const inputSchema = lazySchema(() =>
  z.object({
    file_path: z.string(),
    content: z.string(),
    encoding: z.enum(['utf-8', 'base64']).optional(),
  })
)

// Parallel initialization — total time = slowest operation (~120ms)
async function initWorkspace() {
  const [config, gitStatus, files] = await Promise.all([
    readConfig(),       // 50ms — runs concurrently
    getGitStatus(),     // 120ms — runs concurrently (bottleneck)
    listOpenFiles(),    // 30ms — runs concurrently
  ])
  return { config, gitStatus, files }
}
```

## Signals that you need this pattern

- A new Tool's `inputSchema` is a bare `z.object({...})` at module scope (not wrapped in `lazySchema`)
- Startup time regressed after adding a new module with expensive top-level initialization
- A function called multiple times per conversation (git status, file listing) is not memoized
- Independent async operations in initialization code are awaited sequentially instead of `Promise.all()`-ed
- `feature('FLAG')` is not used for optional large features (assistant mode, voice mode, remote mode)

## Signals that you're over-applying it

- Don't memoize functions whose inputs vary per call — `memoize()` with no cache key returns the same result for all inputs
- Don't use `lazySchema()` for schemas used in tests or utilities — the startup savings are irrelevant there
- Don't parallelize operations that depend on each other's output

## Works with

- `async-concurrency` — Promise.all() for parallel I/O
- `module-organisation` — which modules are lazily loaded
- `types-and-interfaces` — lazySchema() wraps Zod schema definitions
