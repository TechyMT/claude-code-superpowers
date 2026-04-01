---
name: error-handling
description: |
  Teaches Claude Code's error philosophy: tool call() throws typed errors, the framework catches them and formats them for the LLM as structured tool result content, and custom error types carry structured context at system boundaries. Use this whenever writing a tool, utility function, or service that can fail. The distinction between "errors the LLM should see" and "errors that crash the session" is the core insight.
---

# Error Handling

## The pattern

Claude Code has two error categories. **Session errors** are unrecoverable failures that crash the process or abort the current session (network configuration issues, authentication failures, corrupted state). **Tool errors** are expected failures that should be shown to the LLM so it can reason about them, retry, or explain the situation to the user.

The fundamental rule: `tool.call()` throws typed errors for expected failures. The framework catches them, calls `formatError(error)` from `src/utils/toolErrors.ts`, and delivers the formatted string to the LLM as a `tool_result` block with `is_error: true`. Only truly unrecoverable infrastructure failures should propagate past the framework as unhandled exceptions.

## Why this matters

The LLM is in a conversation loop. If a tool throws a raw, unformatted exception, the framework must decide what to show the model — a stack trace is not useful. By throwing typed errors (`ShellError`, `AbortError`, `Error`), tools hand the framework enough structure to produce a clean, readable message. For example, a file-not-found error thrown from `FileReadTool` lets the LLM suggest the correct path or ask the user to check.

This also means the user sees a coherent explanation rather than an error box. The LLM translates the technical error into natural language in its next response.

Custom error types (`ShellError`, `AbortError`) are used at system boundaries — they carry structured context (exit code, stderr, interrupted flag) that `formatError` uses to build the right message.

## How to apply it

1. Wrap all I/O in try-catch inside `call()`. Catch specific error types first, then rethrow or let them propagate.
2. `throw new Error(message)` for expected, non-shell failures. Include enough context for the LLM to act: what was attempted, what failed, what the user could check.
3. For ENOENT (file not found): use `isENOENT()` from `utils/errors.js` to detect it and throw `new Error("File not found: /path. Does it exist?")`.
4. For command failures: throw `new ShellError(stdout, stderr, code, interrupted)` — `formatError` assembles exit code and stderr automatically.
5. For user cancellation: throw `new AbortError()` — `formatError` maps it to a clean interruption message.
6. Define custom error types when an error carries structured data that formatting logic needs.

## In the source

```typescript
// Source: src/tools/BashTool/BashTool.tsx
if (result.preSpawnError) {
  throw new Error(result.preSpawnError)  // pre-spawn failures → plain Error
}
if (interpretationResult.isError && !isInterrupt) {
  throw new ShellError('', outputWithSbFailures, result.code, result.interrupted)
  // ShellError carries structured context; formatError builds the LLM message
}

// Source: src/utils/toolErrors.ts — framework calls this after catching
export function formatError(error: unknown): string {
  if (error instanceof AbortError) {
    return error.message || INTERRUPT_MESSAGE_FOR_TOOL_USE
  }
  if (!(error instanceof Error)) return String(error)
  const parts = getErrorParts(error)
  return parts.filter(Boolean).join('\n').trim() || 'Command failed with no output'
  // truncated to 10 000 chars if longer
}

export function getErrorParts(error: Error): string[] {
  if (error instanceof ShellError) {
    return [`Exit code ${error.code}`, error.interrupted ? '...' : '', error.stderr, error.stdout]
  }
  return [error.message]
}

// Source: src/tools/FileReadTool/FileReadTool.ts
import { isENOENT } from '../../utils/errors.js'  // NOT utils/file.js

} catch (error) {
  if (isENOENT(error)) {
    throw new Error(`File not found: ${file_path}. Does it exist?`)
  }
  throw error  // re-throw unknown errors — framework handles them
}
```

The abort-error branch is non-obvious: when the user presses Ctrl+C, an `AbortError` propagates through all in-flight async calls. Tools must let it propagate (or throw a new `AbortError`) so `formatError` returns a clean "interrupted" message.

## Apply it to your code

**Before** — tool that returns a wrong shape instead of throwing:
```typescript
async call(args, context, canUseTool) {
  try {
    const response = await fetch(args.url)
    if (!response.ok) {
      return { type: 'error', error: `HTTP ${response.status}` }  // Wrong shape: ToolResult has no error variant
    }
    return { type: 'success', data: await response.text() }       // Wrong shape: no 'type' field on ToolResult
  } catch (err) {
    return { type: 'error', error: String(err) }                  // Wrong: swallows AbortError, wrong shape
  }
}
```

**After** — tool that throws typed errors:
```typescript
async call(args, context, canUseTool) {
  try {
    const response = await fetch(args.url, {
      signal: context.abortController.signal,  // WHY: lets AbortError propagate on Ctrl+C
    })

    if (!response.ok) {
      // WHY: throw Error so formatError surfaces a readable message to the LLM
      throw new Error(`HTTP ${response.status} ${response.statusText} fetching ${args.url}`)
    }

    return { data: await response.text() }  // WHY: ToolResult<T> is just { data: T }
  } catch (err) {
    if (err instanceof AbortError) throw err  // WHY: let framework handle cancellation cleanly
    // WHY: network errors (DNS failure, timeout) re-thrown as plain Error with context
    throw new Error(`Network error fetching ${args.url}: ${err instanceof Error ? err.message : String(err)}`)
  }
}
```

## Signals that you need this pattern

- Tool's `call()` returns `{ type: 'error', error: ... }` — there is no error variant on `ToolResult<T>`; throw instead
- Unhandled promise rejections appearing in logs from tool execution
- The LLM receives a raw stack trace as tool result content
- Error messages say "something went wrong" without a file path, exit code, or actionable hint
- `isENOENT` imported from `utils/file.js` — the correct source is `utils/errors.js`
- `AbortError` is caught and swallowed — Ctrl+C produces an error dialog instead of a clean cancellation

## Signals that you're over-applying it

- Utilities called only within other tools don't need special error handling — they can throw freely; the calling tool catches
- Session-level errors (authentication failure, Claude API unreachable) should propagate as exceptions to the session handler, not be swallowed as tool errors
- Don't write exhaustive catch branches for impossible error cases from internal calls you control

## Works with

- `tool-definition` — the ToolResult type and where errors appear in the return value
- `permission-system` — handling permission denial as a specific error case
- `async-concurrency` — AbortError handling when tools are cancelled
