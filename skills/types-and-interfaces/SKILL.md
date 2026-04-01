---
name: types-and-interfaces
description: |
  Teaches Claude Code's type design: discriminated unions for output data shapes, Zod schemas as the single source of truth for tool inputs, DeepImmutable for AppState, and z.infer<> to derive TypeScript types from schemas. Use this when designing new types, adding AppState fields, or defining tool input schemas. The core discipline: define shape once, derive all other representations from it.
---

# Types and Interfaces

## The pattern

Claude Code uses three type patterns consistently: **discriminated unions for output data shapes** (the Zod `outputSchema` for tools that can return one of several kinds of result), **Zod schemas as single source of truth** for tool inputs (the schema drives runtime validation AND TypeScript types simultaneously via `z.infer<>`), and **DeepImmutable wrappers** for AppState to prevent accidental mutation.

The discipline is: define shape once, derive everything else. A Zod schema for a tool's input is defined once in `inputSchema`. The TypeScript type `z.infer<typeof inputSchema>` is derived from it. The LLM's JSON schema is derived from it. Runtime validation uses it. There is no second `interface FooInput` to drift from the schema.

## Why this matters

The LLM produces JSON. JSON must be validated at runtime. TypeScript types catch compile-time issues. If these are defined separately, they drift: the TypeScript interface says `limit?: number` but the runtime validator doesn't check it's a number, leading to a crash at runtime. Zod solves this by being both the runtime validator and the source for `z.infer<>` types.

Discriminated unions for output data types (e.g. a file read that returns text, image, pdf, or unchanged) make exhaustive handling compiler-enforced. The TypeScript compiler narrows the type in each `case` branch. This eliminates an entire class of missing-case bugs. For errors, tools throw rather than returning an error variant — the success/error boundary is the thrown-exception contract, not the `ToolResult` shape.

AppState uses `DeepImmutable<T>` to prevent anyone from mutating state directly. All state changes must go through `setAppState(prev => next)`, making state transitions traceable and atomic.

## How to apply it

1. **Tool inputs**: always use Zod. Use `lazySchema()` wrapper from `utils/lazySchema.ts` to defer parsing until first access (faster startup). Use `z.strictObject` (not `z.object`) to reject unknown keys. Derive the TypeScript type with `type MyInput = z.infer<typeof inputSchema>`.
2. **Tool output shapes**: when a tool can return one of several kinds of result, define a Zod `outputSchema` using `z.discriminatedUnion`. Derive the TypeScript type with `type Output = z.infer<typeof outputSchema>`.
3. **AppState fields**: add to the `AppState` type in `AppStateStore.ts`. The type is `DeepImmutable<{...}>` — nested objects are also deeply immutable. You cannot `.push()` to arrays; you must spread them.
4. **Message types**: use the existing discriminated union. Don't add new message types without understanding the normalization pipeline — messages flow through multiple transformations before reaching the API.
5. **Avoid `any`**: use `unknown` for values whose type is genuinely unknown at the call site, then narrow with type guards.

## In the source

```typescript
// Source: src/Tool.ts (ToolResult — tools return data or throw, no error variant)
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: { _meta?: Record<string, unknown>; structuredContent?: Record<string, unknown> }
}

// Tools signal errors by throwing — not by returning { type: 'error' }.
// A successful call returns { data: result }:
return { data: result }

// Source: src/tools/FileReadTool/FileReadTool.ts (discriminated union for output data)
import { lazySchema } from '../../utils/lazySchema.js'
import { z } from 'zod'

// outputSchema uses discriminatedUnion because a file read can yield several kinds of result
const outputSchema = lazySchema(() =>
  z.discriminatedUnion('type', [
    z.object({
      type: z.literal('text'),
      file: z.object({ filePath: z.string(), content: z.string(), numLines: z.number() }),
    }),
    z.object({
      type: z.literal('image'),
      file: z.object({ filePath: z.string(), /* ... */ }),
    }),
    z.object({
      type: z.literal('pdf'),
      file: z.object({ filePath: z.string(), /* ... */ }),
    }),
    z.object({
      type: z.literal('file_unchanged'),
      file: z.object({ filePath: z.string() }),
    }),
  ])
)
type OutputSchema = ReturnType<typeof outputSchema>
export type Output = z.infer<OutputSchema>

// TypeScript narrows in each branch — no cast needed:
if (output.type === 'text') {
  console.log(output.file.content)  // TypeScript knows content: string here
}

// Source: src/tools/FileReadTool/FileReadTool.ts (Zod inputSchema as single source of truth)
// lazySchema() defers parsing until first access — not evaluated at module load
const inputSchema = lazySchema(() =>
  z.strictObject({  // strictObject rejects unknown keys
    file_path: z.string().describe('The absolute path to the file to read'),
    offset: z.number().int().nonnegative().optional()
      .describe('Line number to start reading from'),
    limit: z.number().int().positive().optional()
      .describe('Maximum number of lines to read'),
  })
)
type InputSchema = ReturnType<typeof inputSchema>
export type Input = z.infer<InputSchema>
// Equivalent to: { file_path: string; offset?: number; limit?: number }

// Source: src/state/AppStateStore.ts (DeepImmutable AppState)
export type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  tasks: Record<string, TaskState>  // DeepImmutable makes this read-only recursively
  // ...
}>

// Mutation is impossible:
// state.verbose = true  // TypeScript error: cannot assign to read-only property

// Correct: derive new state
context.setAppState(prev => ({ ...prev, verbose: true }))

// Source: src/Task.ts (TaskStatus string literal union)
export type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed'

// Exhaustive check in switch is enforced by TypeScript:
function handleStatus(status: TaskStatus) {
  switch (status) {
    case 'pending': return 'pending'
    case 'running': return 'running'
    case 'completed': return 'completed'
    case 'failed': return 'failed'
    case 'killed': return 'killed'
    // No default needed — TypeScript knows all cases are covered
  }
}
```

The `lazySchema()` wrapper deserves attention: Zod schema construction does non-trivial work at parse time (it builds validator functions). Calling it at module load time adds to startup latency. `lazySchema()` wraps the constructor in a thunk evaluated only on first access, which is typically when the first tool call happens — after startup is already complete.

## Apply it to your code

**Before** — separate interface and runtime validation that can drift:
```typescript
// TypeScript type defined separately from runtime validator
interface SearchInput {
  query: string
  maxResults?: number
  caseSensitive?: boolean
}

// Runtime validator defined separately — can drift from the interface above
function validateSearchInput(input: unknown): SearchInput {
  if (typeof input !== 'object') throw new Error('Not an object')
  if (typeof (input as any).query !== 'string') throw new Error('query must be string')
  return input as SearchInput  // Dangerous cast
}
```

**After** — Zod as single source of truth:
```typescript
import { lazySchema } from '../../utils/lazySchema.js'
import { z } from 'zod'

// One definition drives both TypeScript types and runtime validation
const inputSchema = lazySchema(() =>
  z.strictObject({  // rejects unknown keys — use z.strictObject, not z.object
    query: z.string().min(1).describe('Text to search for'),
    maxResults: z.number().int().positive().max(1000).optional()
      .describe('Maximum number of results (default: 100)'),
    caseSensitive: z.boolean().optional()
      .describe('Whether to match case exactly (default: false)'),
  })
)

// Type derived from schema — no separate interface to maintain
type InputSchema = ReturnType<typeof inputSchema>
type SearchInput = z.infer<InputSchema>
// { query: string; maxResults?: number; caseSensitive?: boolean }

// The schema validates AND types the result — no cast needed
const parsed = inputSchema.parse(rawInput)  // throws ZodError if invalid
const safe = inputSchema.safeParse(rawInput)  // returns { success, data } or { success: false, error }
```

## Signals that you need this pattern

- A TypeScript interface and a manual validation function exist side-by-side for the same data shape
- A tool that returns multiple kinds of result has no `outputSchema` with a discriminated union
- AppState is mutated with direct assignment (`state.field = value`) instead of via `setAppState`
- Tool input types are defined as plain TypeScript interfaces rather than derived from Zod schemas
- `as any` or unsafe type casts appear where Zod validation should be used

## Signals that you're over-applying it

- Internal pure-function parameters that are always called with known types don't need Zod validation
- Simple string literal union types (`'left' | 'right'`) don't need Zod schemas — just use the union directly
- Don't wrap every internal helper's return type in a discriminated union; use discriminated unions at system/module boundaries

## Works with

- `tool-definition` — Zod schemas as `inputSchema` and `outputSchema` in the Tool interface
- `error-handling` — tools throw for errors; ToolResult carries success data only
- `domain-model` — AppState's DeepImmutable type shape
