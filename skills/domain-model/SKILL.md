---
name: domain-model
description: |
  Teaches Claude Code's core domain model: Tool, Message, Task, ToolUseContext, AppState, and Command — the six concepts every part of the system is built around. Use this before writing any new tool, command, or feature that touches the conversation loop. Essential for understanding how Claude's capability system is structured and why the abstractions exist.
---

# Domain Model

## The pattern

A well-designed AI agent system separates three concerns: (1) **capability objects** — typed schemas declaring what the agent can do, with permission gates controlling access; (2) **runtime injection** — a context object passed to every capability invocation, carrying state, abort signals, and callbacks (so capabilities are independently testable, no global singletons); (3) **atomic state** — an immutable global store updated via reducer functions, so UI renders consistently and state transitions are traceable.

Claude Code names these: Tool (capability), ToolUseContext (injection), and AppState (state store). The other three concepts — Message, Task, and Command — complete the model: A **Tool** is a named capability with a typed input schema, a permission check, and an execution function. A **Message** is an immutable turn in the conversation — user input, assistant response, or system event. A **Task** is an optional background process that a tool can spawn. A **ToolUseContext** is the runtime environment injected into every tool call — it carries state, abort signals, callbacks, and configuration. **AppState** is the mutable global state store that the UI and tools read from and write to. A **Command** is a user-facing slash command that resolves to either a prompt template (injected into the conversation) or a local function.

Without understanding these six, you will model Claude Code incorrectly — treating it as a plain CLI instead of a React-driven state machine with a structured capability system.

## Why this matters

Claude Code must bridge an LLM (which communicates via structured JSON tool calls) with a local developer environment (file system, shell, git). The Tool abstraction provides a single contract: the LLM produces a `tool_name` + `input` JSON block, the runtime validates the input against a Zod schema, checks permissions, and calls `tool.call()`. The result is returned to the LLM as a tool result block.

The ToolUseContext solves a layering problem: tools need access to state, permissions, callbacks, and configuration without those being global singletons. Every tool call receives the full context, making tools independently testable and the dependency graph explicit.

AppState uses a Zustand-like pattern in a React TUI. State transitions happen atomically via `setAppState(prev => next)` — the same pattern as React's `setState`, but available outside React components via the context object.

## How to apply it

1. When writing a new Tool: use `buildTool({...})` from `src/Tool.ts` with a `get inputSchema()` getter. Define your Zod schema first — it drives the LLM's JSON schema, the validation, and the TypeScript type. Return `{ data: ... }` from `call()`.
2. When writing a new Command: define a `PromptCommand` (for prompt injection) or `LocalCommand` (for local execution) in `src/commands/`. Register it in `src/commands.ts`.
3. When accessing state from a tool: use `context.getAppState()` for reads. Use `context.setAppState(prev => ({...prev, field: newValue}))` for writes. Never use module-level globals.
4. When spawning background work: create a Task via `TaskCreateTool` or spawn a local agent task. Tasks are tracked in `AppState.tasks` and live in disk-backed output files.

## In the source

```typescript
// Source: src/Tool.ts
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  aliases?: string[]
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>
  description(
    input: z.infer<Input>,
    options: {
      isNonInteractiveSession: boolean
      toolPermissionContext: ToolPermissionContext
      tools: Tools
    },
  ): Promise<string>
  readonly inputSchema: Input
  isConcurrencySafe(input: z.infer<Input>): boolean
  isReadOnly(input: z.infer<Input>): boolean
  isDestructive?(input: z.infer<Input>): boolean
  interruptBehavior?(): 'cancel' | 'block'
}
```

`canUseTool` is passed into `call()` rather than called before it — this lets tools that spawn sub-tools forward the permission function to their children. `onProgress` is optional, used for streaming partial results to the UI while the tool runs. `isConcurrencySafe()` and `isReadOnly()` are per-call decisions (they receive the input), not per-tool constants.

## Apply it to your code

**Before** — a tool written as a plain function:
```typescript
async function readFile(path: string, maxLines?: number): Promise<string> {
  const content = await fs.readFile(path, 'utf-8')
  return content
}
```

**After** — same capability as a proper Tool:
```typescript
// Source: src/tools/FileReadTool/FileReadTool.ts
// inputSchema drives LLM JSON schema, validation, and TypeScript types simultaneously
const inputSchema = lazySchema(() =>
  z.strictObject({
    file_path: z.string().describe('Absolute path to the file to read'),
    limit: z.number().int().positive().optional().describe('Max lines to read'),
    offset: z.number().int().nonnegative().optional().describe('Line to start reading from'),
  })
)

export const FileReadTool = buildTool({
  name: FILE_READ_TOOL_NAME,
  // isConcurrencySafe: reading never mutates, safe to run in parallel
  isConcurrencySafe() { return true },
  isReadOnly() { return true },

  async checkPermissions(input, context) {
    return checkReadPermissionForTool(FileReadTool, input, context.getAppState().toolPermissionContext)
  },

  async call({ file_path, limit, offset }, context, _canUseTool?) {
    const content = await readFileInRange(file_path, offset, limit)
    // Return data to the LLM as a structured tool result block
    return { data: { type: 'text', file: { filePath: file_path, content } } }
  },

  async description(input) {
    // Description is shown to the user (not the LLM) in permission prompts
    return `Read ${input.file_path}`
  },

  get inputSchema() { return inputSchema() },
})
```

## Signals that you need this pattern

- You're writing a new capability directly in the query loop or as a side effect of another tool
- You have a utility function that needs permission checking but is called ad-hoc
- You're storing conversation-scoped state in a module-level variable
- A tool is importing AppState directly rather than reading it from `context.getAppState()`
- You're creating a command that needs access to tools but can't find how to wire it in

## Signals that you're over-applying it

- A simple utility function that performs pure computation (path joining, string formatting) does not need to be a Tool
- Not every helper needs ToolUseContext — pass only the fields it actually needs
- Don't create a Task for work that completes in milliseconds; Tasks are for long-running or interruptible work

## Works with

- `tool-definition` — the complete guide to implementing a Tool correctly
- `permission-system` — how `canUseTool` works and how to define permission rules
- `task-system` — how to spawn and track background work from a Tool
- `async-concurrency` — how `isConcurrencySafe` affects parallel execution
