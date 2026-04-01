---
name: tool-definition
description: |
  Teaches the complete pattern for defining a Claude Code Tool: schema-first design with Zod, the buildTool factory for safe defaults, checkPermissions for side-effect gates, progress callbacks for streaming, and returning { data: T } from call(). Use this whenever adding a new tool to Claude Code or when modifying how an existing tool handles input, permissions, or output. The pattern is the same whether the tool reads files, executes bash, or spawns subagents.
---

# Tool Definition

## The pattern

A Tool is defined by calling `buildTool({ ... })`, which fills in safe defaults for `isEnabled`, `isConcurrencySafe`, `isReadOnly`, `isDestructive`, `checkPermissions`, and other housekeeping methods. The Input is a Zod schema exposed via a `get inputSchema()` getter. The Output is the TypeScript type of `data` inside the result. These compose: the Zod schema drives runtime validation AND TypeScript types simultaneously, eliminating a class of bugs where validation and types diverge.

The key insight: **buildTool factory + checkPermissions method + typed throw for errors**. Permissions are not checked inside `call()` — the framework calls `checkPermissions(input, context)` before dispatching to `call()`. Errors are thrown (not returned), and the framework formats them for the LLM.

## Why this matters

The LLM produces tool calls as JSON objects. Claude Code must: (1) validate the JSON against a schema, (2) check if the user permits this call, (3) execute the tool, (4) return a structured result back to the LLM. These four steps must be explicit and in order.

If a tool checked permissions after execution (or not at all), security would be broken. If validation happened after permission checking, the LLM could construct inputs that bypass validation for permitted tools. The pattern enforces the correct order: schema validation happens automatically when the framework calls the tool; `checkPermissions` is called by the framework before `call()` is ever reached; `call()` can assume permission is already granted.

Using Zod as the schema source of truth means the LLM's JSON schema (shown in the system prompt) and the runtime validator are the same artifact. When you change one, you change both.

## How to apply it

1. Define `inputSchema` using `lazySchema(() => z.strictObject({ ... }))`. Add `.describe()` to each field — these become the LLM's parameter documentation.
2. Implement `isConcurrencySafe(input)` and `isReadOnly(input)`. These are called with the actual input, so you can make them input-dependent (e.g., a bash tool is concurrent-safe only for read commands).
3. Implement `checkPermissions()` for side-effect tools — the framework calls it before `call()`. Return a `PermissionDecision` indicating whether execution is allowed.
4. Return `{ data: result }` from `call()` on success; throw for errors (the framework formats thrown errors for the LLM).
5. Use `onProgress` to stream partial results to the UI. Call it with typed progress objects as work proceeds.

## In the source

```typescript
// Source: src/tools/FileReadTool/FileReadTool.ts (real file, real pattern)
import { buildTool, type ToolDef } from '../../Tool.js'
import { lazySchema } from '../../utils/lazySchema.js'

const inputSchema = lazySchema(() =>
  z.strictObject({
    file_path: z.string().describe('The absolute path to the file to read'),
    offset: z.number().int().nonnegative().optional().describe('Line number to start reading from'),
    limit: z.number().int().positive().optional().describe('Number of lines to read'),
  })
)
type InputSchema = ReturnType<typeof inputSchema>

export const FileReadTool = buildTool({
  name: FILE_READ_TOOL_NAME,
  searchHint: 'read files, images, PDFs, notebooks',
  strict: true,
  async description() { return DESCRIPTION },

  // Zod schema exposed as a getter — evaluated lazily on first access
  get inputSchema(): InputSchema { return inputSchema() },

  isConcurrencySafe() { return true },
  isReadOnly() { return true },

  // Framework calls this BEFORE call() — tool never needs to re-check
  async checkPermissions(input, context): Promise<PermissionDecision> {
    return checkReadPermissionForTool(FileReadTool, input, appState.toolPermissionContext)
  },

  async call({ file_path, offset = 1, limit }, context, _canUseTool?) {
    // Permission already granted by the time we reach here
    const content = await readFile(file_path, { offset, limit })
    // Return { data: T } on success — throw on error
    return { data: { type: 'text', file: { filePath: file_path, content, numLines: limit } } }
  },
})
```

Notice that `description()` is async and returns a string rather than a static property — this lets the UI show "Read src/main.tsx" instead of "Read a file". `_canUseTool` in `call()` is available only for forwarding to sub-tools; most tools ignore it entirely.

## Apply it to your code

**Before** — capability implemented as an unstructured function:
```typescript
// No schema, no permission check, throws on error, no streaming
async function searchFiles(pattern: string, dir: string): Promise<string[]> {
  const results = await glob(pattern, { cwd: dir })
  if (results.length === 0) {
    throw new Error(`No files matching ${pattern}`)
  }
  return results
}
```

**After** — same capability as a proper Tool:
```typescript
import { z } from 'zod'
import { buildTool } from '../../Tool.js'
import { lazySchema } from '../../utils/lazySchema.js'

const inputSchema = lazySchema(() =>
  z.strictObject({
    pattern: z.string().describe('Glob pattern to match (e.g. **/*.ts)'),
    path: z.string().optional().describe('Directory to search in. Defaults to cwd.'),
  })
)
type InputSchema = ReturnType<typeof inputSchema>

export const GlobTool = buildTool({
  name: GLOB_TOOL_NAME,
  searchHint: 'find files by glob pattern',
  async description(input) {
    return `Find files matching ${input.pattern}${input.path ? ` in ${input.path}` : ''}`
  },

  get inputSchema(): InputSchema { return inputSchema() },

  // Glob is pure read — safe to run concurrently with other read operations
  isConcurrencySafe: () => true,
  isReadOnly: () => true,

  // Read-only tools typically need no permission check; buildTool supplies a permissive default.
  // For a tool with side effects, implement checkPermissions() here instead.

  async call(args, context) {
    // By the time call() runs, permissions are already settled
    const cwd = args.path ?? context.getAppState().cwd
    const results = await glob(args.pattern, { cwd })
    // Return empty array on no matches — not an error; let the LLM decide
    // Throw on unexpected failures — framework formats the error for the LLM
    return { data: results }
  },
})
```

## Signals that you need this pattern

- A capability needs LLM access but is implemented as a function called from the query loop directly
- A tool is reading `process.env` or global singletons instead of `context.getAppState()`
- A tool manually calls `canUseTool` inside `call()` instead of implementing `checkPermissions()`
- The Zod schema and the TypeScript interface are defined separately and can drift
- `isConcurrencySafe` always returns `false` without examining the input

## Signals that you're over-applying it

- Pure computation helpers (formatting, parsing) do not need Tool wrapping
- Utilities called only from tests or other tools' `call()` implementations don't need schemas
- Don't implement `isDestructive` for every tool — only tools that perform irreversible mutations

## Works with

- `domain-model` — explains where Tools fit in the full system
- `permission-system` — how `checkPermissions` evaluates and what happens after denial
- `async-concurrency` — how `isConcurrencySafe` affects the execution scheduler
- `error-handling` — full error propagation philosophy for tool results
