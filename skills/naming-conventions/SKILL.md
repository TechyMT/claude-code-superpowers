---
name: naming-conventions
description: |
  Teaches Claude Code's naming system: Tool files are [Name]Tool.ts[x], commands use kebab-case, types are PascalCase, functions start with verbs in camelCase, booleans use is/has prefixes, constants use UPPER_SNAKE_CASE, and event handlers use on[Event]. Use this when naming new files, types, functions, or variables to match the existing codebase style. Consistent naming is how contributors find code without searching.
---

# Naming Conventions

## The pattern

Claude Code uses a set of naming conventions that encode the role of a file or identifier in its name. A file named `BashTool.tsx` is a Tool implementation with UI concerns. A file named `BashTool.ts` is a Tool implementation with no UI. A function named `isConcurrencySafe` returns a boolean about concurrency. A function named `getBranch` fetches a value. A constant named `MAX_STATUS_CHARS` is a numeric limit. These are not accidental — reading the name tells you the role without reading the implementation.

## Why this matters

At ~512K lines across 1,884 files, Claude Code is too large for any individual to hold in memory. Naming conventions serve as a file-system-level type system: you can predict what's in a file before opening it. `AgentTool/AgentTool.tsx` will export a `Tool<Input, Output>` that has UI progress output. `state/AppStateStore.ts` will export the `AppState` type and initial state. `hooks/useCanUseTool.ts` will export a React hook returning a permission-checking function.

## How to apply it

**Files:**
- Tools: `src/tools/[Name]Tool/[Name]Tool.ts[x]` — `.tsx` if the tool renders JSX output
- Commands: `src/commands/[kebab-name]/index.ts` or `src/commands/[kebab-name].ts`
- React hooks: `src/hooks/use[Name].ts` — always prefixed with `use`
- React components: `src/components/[Name].tsx`
- Utilities: `src/utils/[name].ts` — lowercase, describe the domain (e.g., `git.ts`, `path.ts`)
- Types: `src/types/[name].ts` — lowercase file, PascalCase exports

**Identifiers:**
- Types, interfaces, classes: `PascalCase` (e.g., `ToolUseContext`, `AppState`, `PermissionResult`)
- Functions: `camelCase` starting with a verb (e.g., `createTaskStateBase`, `normalizeMessages`, `getGitStatus`)
- Booleans and boolean-returning functions: `is`/`has` prefix (e.g., `isConcurrencySafe`, `isReadOnly`, `hasWorktreeChanges`, `isENOENT`)
- Event handlers and callbacks: `on[Event]` prefix (e.g., `onProgress`, `onChangeAppState`, `onCompactProgress`)
- Constants: `UPPER_SNAKE_CASE` for numeric/string limits; `PascalCase` for object constants (e.g., `MAX_STATUS_CHARS`, `IDLE_SPECULATION_STATE`)
- Tool exports: named constant matching the file name (e.g., `export const BashTool`, `export const FileReadTool`)
- Async functions: same naming as sync — the `Promise` return type is the indicator, not a naming convention

## In the source

```typescript
// Source: src/tools/BashTool/BashTool.tsx
// File name: [Name]Tool.tsx — Tool with JSX rendering
// Export name: matches file (BashTool)
export const BashTool: Tool<typeof inputSchema> = {
  // Boolean method: is prefix
  isConcurrencySafe(input) { ... },
  isReadOnly(input) { ... },
  isDestructive(input) { ... },

  // Verb prefix: call, description
  async call(args, context, canUseTool) { ... },
  async description(input, options) { ... },
}

// Source: src/context.ts
// Constant: UPPER_SNAKE_CASE for a numeric limit
const MAX_STATUS_CHARS = 10_000

// Function: verb prefix (get)
export const getGitStatus = memoize(async (): Promise<string | null> => { ... })
export const getBranch = async (): Promise<string> => { ... }
export const getIsGit = async (): Promise<boolean> => { ... }

// Source: src/state/AppStateStore.ts
// Type: PascalCase
export type AppState = DeepImmutable<{
  // Boolean fields: is prefix
  isBriefOnly: boolean
  kairosEnabled: boolean
  replBridgeConnected: boolean
  // Object constant: PascalCase
  expandedView: 'none' | 'tasks' | 'teammates'
}>

// Source: src/Tool.ts
// Callback types: on prefix
export type ToolCallProgress<P> = (progress: P) => void
// Used as: onProgress, onChangeAppState, onCompactProgress

// Source: src/hooks/useCanUseTool.ts
// React hook: use prefix
export function useCanUseTool(): CanUseToolFn { ... }

// Source: src/utils/file.ts
// Boolean helper: is prefix
export function isENOENT(err: unknown): boolean { ... }
```

The `getIsGit` naming is notable: it returns `Promise<boolean>`, so it uses the `get` verb prefix (because it fetches from disk/git) combined with the boolean it returns. This is consistent: `getX` fetches, `isX` describes a property — when a function does both, `get` takes precedence.

## Apply it to your code

**Before** — inconsistent naming:
```typescript
// Wrong: file named searchHelper.ts in tools/
// Wrong: function named checkIfSearch instead of isSearch
// Wrong: constant named maxResults instead of MAX_RESULTS
// Wrong: callback named handleProgress instead of onProgress

export function checkIfSearch(command: string): boolean { ... }
const maxResults = 100
const handleProgress = (data: ProgressData) => { ... }
```

**After** — names that match the Claude Code conventions:
```typescript
// File: src/utils/bash.ts (utility, lowercase, domain-named)
// File: src/tools/SearchTool/SearchTool.ts (Tool, PascalCase + Tool suffix)

export function isSearchCommand(command: string): boolean { ... }  // is prefix for boolean
const MAX_RESULTS = 100                                             // UPPER_SNAKE_CASE for limit
const onProgress = (data: ProgressData) => { ... }                 // on prefix for callback
```

## Signals that you need this pattern

- A new Tool file is named `searchHelper.ts` instead of `SearchTool.ts`
- A boolean-returning function is named `checkPermission` instead of `hasPermission` or `isPermissionGranted`
- An event callback is named `handleProgress` or `progressCallback` instead of `onProgress`
- Constants like `maxTokens` or `defaultTimeout` don't use `UPPER_SNAKE_CASE`
- A React hook doesn't start with `use`

## Signals that you're over-applying it

- Single-letter variables in tight loops (`i`, `j`, `k`) are fine — don't verbose-name loop indices
- Test helper functions in test files don't need to follow production naming as strictly
- The `is` prefix for boolean-returning functions is the most important convention; the others are softer

## Works with

- `module-organisation` — naming conventions for files extend to directory structure
- `tool-definition` — canonical names for Tool methods (`call`, `description`, `inputSchema`)
- `types-and-interfaces` — PascalCase type naming
