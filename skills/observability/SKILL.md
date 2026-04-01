---
name: observability
description: |
  Teaches Claude Code's logging and instrumentation conventions: logForDebugging for local structured debug output, logEvent for telemetry analytics events (with PII-safe metadata types), profileCheckpoint() for startup timing, token cost tracking via cost-tracker.ts, and the distinction between diagnostic logs (never shown to users) and tool result content (shown to the LLM). Use this when adding logging, tracking performance, or deciding what to instrument in a new tool or service.
---

# Observability

## The pattern

Claude Code has a strict separation between **diagnostic logs** (structured, PII-safe, written to log files) and **tool result content** (what the LLM and user see). For local debug output, use `logForDebugging(message, data?)` from `src/utils/debug.ts`. For analytics telemetry events sent to Anthropic (with user consent), use `logEvent('event_name', metadata)` from `src/services/analytics/index.ts`. Tool results are returned as strings in `ToolResult.data` or `ToolResult.error`.

The PII-safety contract is enforced at the type level: analytics metadata must be typed as `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` — the suffix is a code-review reminder that the values you are passing are not file contents, user input, or anything identifying.

Startup performance is tracked with `profileCheckpoint(label)` — a lightweight timer that records the time since the previous checkpoint. Token usage is tracked in `cost-tracker.ts`, which accumulates input/output tokens across turns and provides totals for the UI's cost display.

## Why this matters

Claude Code runs on user machines and sends analytics data to Anthropic (with user consent). The `_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` suffix on the metadata type is a reminder enforced by code review: log operation types, timings, and error types — but never file contents, user input, command arguments, or anything that could identify the user or their work.

Startup profiling exists because startup latency is user-visible. A regression in `imports_loaded` timing (before any user code runs) tells engineers which module import caused the slowdown without needing to reproduce locally.

Token cost tracking is a user feature (the session cost display) and an internal constraint (auto-compaction triggers when context approaches the limit). Both consumers use the same accumulator.

## How to apply it

1. Track operation start/end times with `logEvent('operation_completed', { duration_ms } as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS)`. Include a duration measurement.
2. Log error types (not error messages) — error messages often contain user data.
3. Use event names as snake_case strings: `'git_status_started'`, `'tool_call_failed'`, `'mcp_reconnected'`.
4. Use `logForDebugging('message', { context })` for local-only debug output that should not go to telemetry.
5. Add `profileCheckpoint('label')` at natural initialization boundaries.
6. Track token usage via `cost-tracker.ts` — don't implement a second accumulator.
7. Never include in telemetry: file contents, user input, command arguments, file paths, API responses, or any string that might contain user data.

## In the source

```typescript
// Source: src/services/analytics/index.ts + src/tools/BashTool/BashTool.tsx (logEvent pattern)
import { logEvent } from '../../services/analytics/index.js'

// After a bash command completes:
logEvent('tengu_bash_tool_command_executed', {
  command_type: commandType as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  stdout_length: stdout.length,
  exit_code: result.code,
  interrupted: wasInterrupted
})

// Source: src/utils/debug.ts + src/services/tools/toolExecution.ts (logForDebugging pattern)
import { logForDebugging } from '../../utils/debug.js'

export const getGitStatus = memoize(async (): Promise<string | null> => {
  const startTime = Date.now()
  logForDebugging('git_status_started')  // Local debug log only — never sent to telemetry

  const isGit = await getIsGit()
  if (!isGit) {
    logForDebugging('git_status_skipped_not_git', {
      duration_ms: Date.now() - startTime,
    })
    return null
  }

  try {
    // ... git commands ...
    logEvent('git_status_completed', {
      duration_ms: Date.now() - startTime,
    } as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS)
    return formattedStatus
  } catch (error) {
    // Log error TYPE, not error message (which might contain paths)
    logEvent('git_status_failed', {
      duration_ms: Date.now() - startTime,
      // error_type: error instanceof GitError ? 'git_error' : 'unknown'
      // NOT: error.message (could contain file paths or user data)
    } as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS)
    logForDebugging('git_status_error_detail', { message: String(error) })  // Full detail stays local
    return null
  }
})

// Source: src/main.tsx (profileCheckpoint for startup timing)
profileCheckpoint('process_start')
// ... early imports ...
profileCheckpoint('imports_loaded')
await initializeConfig()
profileCheckpoint('config_loaded')
await Promise.all([prefetchKeychain(), connectMCPServers()])
profileCheckpoint('parallel_init_done')
// Each checkpoint records elapsed ms since the previous one

// Source: src/cost-tracker.ts (token accumulation)
export class CostTracker {
  private inputTokens = 0
  private outputTokens = 0

  addUsage(usage: { input_tokens: number; output_tokens: number }): void {
    this.inputTokens += usage.input_tokens
    this.outputTokens += usage.output_tokens
  }

  getTotalCost(): number {
    // Price per token from model config
    return (this.inputTokens * INPUT_COST_PER_TOKEN) +
           (this.outputTokens * OUTPUT_COST_PER_TOKEN)
  }

  getUsageSummary(): string {
    return `${this.inputTokens.toLocaleString()} in / ${this.outputTokens.toLocaleString()} out`
  }
}
```

The distinction between `logEvent` and `logForDebugging` is important: `logEvent` goes to Anthropic's analytics pipeline (with user consent) and must only carry PII-safe metadata. `logForDebugging` writes to a local debug log file only — it is safe to include richer context there.

## Apply it to your code

**Before** — logging user data with console.log, no duration tracking:
```typescript
async call(args, context, canUseTool) {
  console.log(`Reading file: ${args.file_path}`)  // Wrong: logs file path (user data)
  
  try {
    const content = await readFile(args.file_path)
    console.log(`Read ${content.length} bytes`)  // Could expose content size (metadata leakage)
    return { type: 'success', data: content }
  } catch (err) {
    console.error(`Failed: ${err.message}`)  // Wrong: error message may contain file path
    return { type: 'error', error: err.message }
  }
}
```

**After** — PII-safe analytics event with duration, local debug log for detail:
```typescript
async call(args, context, canUseTool) {
  const startTime = Date.now()
  logForDebugging('file_read_started', { path: args.file_path })  // Local only — path stays off telemetry

  const permission = await canUseTool(FileReadTool, args, context)
  if (!permission.granted) {
    logEvent('file_read_permission_denied', {} as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS)
    return { type: 'error', error: permission.reason }
  }

  try {
    const content = await readFile(args.file_path)
    logEvent('file_read_completed', {
      duration_ms: Date.now() - startTime,
      // bytes_read: content.length  — be careful: even file sizes can be identifying
    } as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS)
    return { type: 'success', data: content }
  } catch (err) {
    logEvent('file_read_failed', {
      duration_ms: Date.now() - startTime,
      is_enoent: isENOENT(err),  // Error TYPE is safe — not the message
    } as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS)
    logForDebugging('file_read_error_detail', { message: String(err) })  // Full error stays local
    return { type: 'error', error: formatFileError(err, args.file_path) }
  }
}
```

## Signals that you need this pattern

- `console.log()` calls in tool implementations log file paths, command arguments, or user input
- A new tool has no diagnostic logging at all — operation failures are silent in telemetry
- Startup time regressed and there are no `profileCheckpoint` markers to narrow it down
- Token usage is tracked in a local variable instead of the shared `CostTracker`
- Error logging includes `err.message` in the analytics payload (not just locally)

## Signals that you're over-applying it

- Don't add `logEvent` to pure utilities — only to I/O operations and cross-boundary calls
- Don't track every sub-operation; profile at natural boundaries (initialization phases, tool call start/end)
- Test code doesn't need diagnostic logging

## Works with

- `error-handling` — logForDebugging vs. logEvent distinction
- `hot-paths` — profileCheckpoint() for startup performance
- `system-boundaries` — each boundary module should log its operation outcomes
