---
name: build-tool-factory
description: |
  Teaches how to create a new Claude Code tool using the buildTool() factory from src/Tool.ts. 
  Use this whenever adding a new capability to Claude Code — buildTool is the only correct entry 
  point. It fills in safe defaults for isConcurrencySafe, isReadOnly, isDestructive, 
  checkPermissions, and userFacingName so you only implement what your tool actually needs.
---

# buildTool() Factory

## The pattern

Factory functions for capability objects solve a common problem: a typed interface has many methods, but any given implementation only needs to override a few. Listing all methods explicitly creates noise and makes the "what does this tool actually do" harder to see. A factory fills in safe defaults and lets the author focus on the non-trivial fields.

The pattern has three parts: (1) a minimal definition type (`ToolDef`) where defaultable methods are optional, (2) a factory (`buildTool`) that merges your definition with safe defaults, and (3) lazy schema getters so schemas are built on first access, not at module load time.

```typescript
// DefaultableToolKeys — methods buildTool fills in with safe defaults
type DefaultableToolKeys =
  | 'isEnabled'
  | 'isConcurrencySafe'
  | 'isReadOnly'
  | 'isDestructive'
  | 'checkPermissions'
  | 'toAutoClassifierInput'
  | 'userFacingName'

// ToolDef — what you pass to buildTool (same as Tool but defaultable methods are optional)
export type ToolDef<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = Omit<Tool<Input, Output, P>, DefaultableToolKeys> &
  Partial<Pick<Tool<Input, Output, P>, DefaultableToolKeys>>

// buildTool fills in safe defaults for all DefaultableToolKeys
export function buildTool<D extends ToolDef>(def: D): BuiltTool<D>
```

| Default key | Safe default value |
|---|---|
| `isConcurrencySafe` | `() => false` |
| `isReadOnly` | `() => false` |
| `isDestructive` | `() => false` |
| `checkPermissions` | `{ behavior: 'allow' }` |
| `isEnabled` | `() => true` |
| `userFacingName` | derived from `name` |

## Why this matters

Claude Code has ~30 tools. Each shares the same set of safety defaults: assume not concurrent-safe, assume not read-only, assume not destructive, assume checkPermissions passes through. Without a factory, every new tool must explicitly declare all of these even when the intent is "no override". The factory encodes the security-safe posture as the default: new tools are restrictive until explicitly opted out.

The alternative — `export const X: Tool<typeof inputSchema> = { ... }` — forces the author to repeat every defaultable method for each tool. It also makes new tools opt-in to safety (you must explicitly set `isDestructive: () => false`) rather than opt-out (the factory sets it for you and you override only when needed).

## How to apply it

1. Create `src/tools/[Name]Tool/[Name]Tool.ts` (or `.tsx` if the tool renders JSX output)
2. Define `inputSchema` with `lazySchema(() => z.strictObject({...}))` — use `z.strictObject` to reject unknown keys
3. Define `outputSchema` the same way when the tool returns structured data the LLM needs to inspect
4. Call `buildTool({ name, inputSchema getter, call, ... })` — only override methods that differ from the defaults
5. Override `isConcurrencySafe` and `isReadOnly` if your tool is side-effect-free (defaults are `false`)
6. Implement `checkPermissions` if your tool accesses files, shell, or network
7. Register the tool in `src/tools.ts`

## In the source

```typescript
// Source: src/tools/FileReadTool/FileReadTool.ts
import { buildTool, type ToolDef } from '../../Tool.js'
import { lazySchema } from '../../utils/lazySchema.js'
import { z } from 'zod/v4'

const inputSchema = lazySchema(() =>
  z.strictObject({
    file_path: z.string().describe('Absolute path to the file'),
    limit: z.number().int().positive().optional().describe('Max lines'),
  })
)
type InputSchema = ReturnType<typeof inputSchema>
export type Input = z.infer<InputSchema>

export const FileReadTool = buildTool({
  name: FILE_READ_TOOL_NAME,
  searchHint: 'read files, images, PDFs, notebooks',  // helps ToolSearch find deferred tools
  strict: true,  // rejects unknown input fields

  async description() { return 'Read a file' },

  // Lazy getter — inputSchema() is not called at module load time
  get inputSchema(): InputSchema { return inputSchema() },

  // Overrides the false default — this tool is safe to run concurrently
  isConcurrencySafe() { return true },
  // Overrides the false default — this tool makes no mutations
  isReadOnly() { return true },

  // checkPermissions is called by the framework BEFORE call() — not inside call()
  async checkPermissions(input, context): Promise<PermissionDecision> {
    return checkReadPermissionForTool(FileReadTool, input, context.getAppState().toolPermissionContext)
  },

  async call({ file_path, limit }, context, _canUseTool?) {
    const content = await readFileInRange(file_path, 1, limit)
    return { data: { type: 'text' as const, file: { filePath: file_path, content } } }
  },
})
```

Notice `isConcurrencySafe` and `isReadOnly` are the only defaultable methods overridden here — everything else (isDestructive, isEnabled, userFacingName, toAutoClassifierInput) is left to `buildTool`'s safe defaults.

## Apply it to your code

**Before** — a new capability added as a plain function (no schema, no factory, no lazy schema):

```typescript
// No schema, no permission check, no lazy loading
async function summariseAsset(assetId: string, depth: number): Promise<string> {
  const asset = await fetchAsset(assetId)
  return buildSummary(asset, depth)
}
```

**After** — same capability as a proper `buildTool({...})` with `lazySchema`, `checkPermissions`, and `{ data: result }` return:

```typescript
import { buildTool } from '../../Tool.js'
import { lazySchema } from '../../utils/lazySchema.js'
import { z } from 'zod/v4'

const inputSchema = lazySchema(() =>
  z.strictObject({
    asset_id: z.string().describe('Qualified name or GUID of the asset'),
    depth: z.number().int().min(1).max(3).optional().default(1)
      .describe('How many relationship hops to include'),
  })
)
type InputSchema = ReturnType<typeof inputSchema>

export const SummariseAssetTool = buildTool({
  name: SUMMARISE_ASSET_TOOL_NAME,
  async description() { return 'Summarise an Atlan asset and its relationships' },

  get inputSchema(): InputSchema { return inputSchema() },

  // Read-only network call — safe to run concurrently
  isConcurrencySafe: () => true,
  isReadOnly: () => true,

  async checkPermissions(input, context) {
    return checkNetworkPermission(SummariseAssetTool, input, context)
  },

  async call({ asset_id, depth }, context) {
    const asset = await fetchAsset(asset_id, context.getAppState().atlanClient)
    const summary = buildSummary(asset, depth)
    return { data: { summary, assetId: asset_id } }
  },
})
```

## Signals that you need this pattern

- A new tool is defined as `export const X: Tool<typeof inputSchema> = { ... }` (direct interface implementation — bypasses the factory and forces explicit defaults)
- A new tool's `inputSchema` is a bare `z.object({...})` at module scope instead of wrapped in `lazySchema()` — schema is built at import time, hurting startup
- A new tool implements all 7 defaultable methods explicitly when most are just the default value
- A new tool uses `z.object` instead of `z.strictObject` — unknown input fields will silently pass through validation

## Signals that you're over-applying it

- Don't use `buildTool` for test helpers or mock tools — it's for production tools registered in `tools.ts`
- Don't add `outputSchema` for tools that return plain strings or simple unstructured data
- `searchHint` is only needed for large tools likely to be deferred by ToolSearch; small tools don't need it

## Works with

- `tool-definition` — the full Tool interface and what each method does
- `permission-system` — implementing `checkPermissions` for tools with side effects
- `hot-paths` — why `lazySchema()` matters for startup performance
- `module-organisation` — where to put the new tool file and how to register it
