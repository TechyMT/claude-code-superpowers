---
name: async-concurrency
description: |
  Teaches Claude Code's concurrency model: isConcurrencySafe() for parallel tool execution, AbortController propagation for cancellation, Promise.all() for parallel initialization, and the StreamingToolExecutor for concurrent progress streams. Use this when writing tools that could benefit from parallel execution, need to respect cancellation, or must stream partial results. Read-only tools that don't declare concurrency safety are needlessly serialized.
---

# Async Concurrency

## The pattern

Claude Code's tool execution is mostly sequential — one tool at a time — but read-only tools can run in parallel when `isConcurrencySafe()` returns true. Cancellation is propagated via `AbortController` passed through `ToolUseContext`. Parallel initialization uses `Promise.all()`. Streaming partial results uses `onProgress` callbacks fed into a `StreamingToolExecutor`.

The key design: concurrency safety is a per-call decision, not a per-tool constant. A bash tool reading `git status` is safe to run concurrently. The same bash tool running `npm install` is not. The tool knows its own input and can make the right call.

## Why this matters

In a developer session, the LLM often requests several read operations in sequence (read file A, read file B, check git status). If these were strictly sequential, the user would wait for three round-trips to disk. `isConcurrencySafe: true` tells the scheduler it can run these in parallel, reducing perceived latency.

Cancellation is critical for long-running tools (bash commands, web fetches). When the user presses Ctrl+C, the AbortController's signal fires and all in-flight tool calls must stop promptly. A tool that ignores `context.abortController.signal` will keep running after the user cancelled, blocking the next turn.

Git status is a concrete example: Claude Code parallelizes five git commands — branch, default branch, status, recent log, and user name — with a single `Promise.all()`. This collapses what would be ~500ms of sequential I/O into a single ~100ms concurrent batch.

## How to apply it

1. For `isConcurrencySafe(input)`: return `true` only for operations that don't mutate state. File reads, globs, searches, and git status commands are safe. File writes, bash mutations, and agent spawns are not.
2. For `isReadOnly(input)`: return `true` for operations that make no persistent changes. Used to determine if the tool can be skipped in speculation mode.
3. Pass `context.abortController.signal` to all async I/O: `fetch()`, `execFile()`, file streams. This ensures immediate cancellation on user interrupt.
4. Check `signal.aborted` at checkpoints in long-running loops.
5. When a tool runs multiple I/O operations that are independent, use `Promise.all()` — don't await them sequentially.
6. For streaming partial results, call `onProgress(progressData)` as work proceeds. The UI subscribes to these calls to show live output.

## In the source

```typescript
// Source: src/context.ts (parallel git status — five concurrent commands)
export const getGitStatus = memoize(async (): Promise<string | null> => {
  const isGit = await getIsGit()
  if (!isGit) return null

  // Five independent git commands run in parallel with Promise.all
  // Sequential would take ~500ms; parallel takes ~100ms
  const [branch, mainBranch, status, log, userName] = await Promise.all([
    getBranch(),
    getDefaultBranch(),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'status', '--short']),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'log', '--oneline', '-n', '5']),
    execFileNoThrow(gitExe(), ['config', 'user.name']),
  ])

  // ... format and return
})

// Source: src/Tool.ts (isConcurrencySafe is input-dependent for BashTool)
export const BashTool: Tool<typeof inputSchema> = {
  isConcurrencySafe(input) {
    // Only safe if the command is classified as search/read
    const classification = isSearchOrReadBashCommand(input.command)
    return classification.isSearch || classification.isRead
  },

  isReadOnly(input) {
    const classification = isSearchOrReadBashCommand(input.command)
    return classification.isSearch || classification.isRead
  },

  async call(args, context, canUseTool) {
    // AbortController signal passed to shell execution
    const result = await executeShell(args.command, {
      signal: context.abortController.signal,
      cwd: getCwd(),
    })
    return { type: 'success', data: result.stdout }
  },
}

// Source: src/Tool.ts (onProgress for streaming output)
async call(args, context, canUseTool, parentMessage, onProgress) {
  const proc = spawnProcess(args.command)

  proc.stdout.on('data', (chunk: Buffer) => {
    // Stream partial output to UI as it arrives
    onProgress?.({
      type: 'output',
      content: chunk.toString(),
    })
  })

  const exitCode = await proc.exitCode
  return { type: 'success', data: proc.collectedOutput }
}
```

The `memoize()` on `getGitStatus` is subtle: git status is called at conversation start and might be called again if a tool triggers re-evaluation. Memoizing prevents running all five git commands twice. The cache is cleared when state changes (e.g., branch switch).

## Apply it to your code

**Before** — read-only tool that blocks concurrent execution unnecessarily:
```typescript
export const SearchTool: Tool<typeof inputSchema> = {
  // Wrong: returns false for a read-only operation, forcing serialization
  isConcurrencySafe: (_input) => false,
  isReadOnly: (_input) => false,

  async call(args, context, canUseTool) {
    // No signal passed — ignores cancellation
    const results = await searchFiles(args.pattern, args.directory)
    return { data: results }
  },
}
```

**After** — tool correctly declares concurrency and respects cancellation:
```typescript
import { AbortError } from '../../utils/errors.js'

export const SearchTool: Tool<typeof inputSchema> = {
  // Search is always read-only — safe to run in parallel with other searches
  isConcurrencySafe: (_input) => true,
  isReadOnly: (_input) => true,

  async call(args, context, canUseTool, _parentMessage, onProgress) {
    const results: string[] = []

    for await (const match of streamSearchResults(args.pattern, args.directory)) {
      // Check cancellation at each result — don't process after user cancelled
      if (context.abortController.signal.aborted) {
        throw new AbortError('Search was cancelled.')
      }

      results.push(match)

      // Stream partial results to UI for responsive feedback
      onProgress?.({ type: 'partial_results', count: results.length })
    }

    return { data: results }
  },
}
```

## Signals that you need this pattern

- A read-only tool (file read, glob, grep) has `isConcurrencySafe: () => false`
- Long-running tools don't pass `context.abortController.signal` to their I/O operations
- A tool makes 3+ independent async calls sequentially when they could be `Promise.all()`-ed
- User reports that Ctrl+C doesn't stop a tool and the session hangs
- The UI shows no progress during a long operation even though partial results are available

## Signals that you're over-applying it

- Mutations (writes, spawns, deletes) must never return `isConcurrencySafe: true` — there are no legitimate parallel-mutation patterns here
- Don't use `Promise.all()` for operations that depend on each other's output
- `onProgress` is optional — don't emit progress for instant operations; the overhead is not worth it

## Works with

- `tool-definition` — where `isConcurrencySafe` and `isReadOnly` are declared
- `error-handling` — handling AbortError from cancelled async operations
- `hot-paths` — parallel initialization as a startup performance pattern
