---
name: permission-system
description: |
  Teaches Claude Code's multi-layered permission system: permission modes (default/bypass/auto), the checkPermissions method on tool definitions, per-tool rules, and how Zod validation gates permission checking. Use this whenever adding a tool that performs side effects, writing permission rules for settings, or understanding why a tool call was denied. Especially important for tools that touch the file system, shell, or network.
---

# Permission System

## The pattern

Every tool call in Claude Code passes through a permission gate before execution. The gate is a `checkPermissions(input, context)` method defined on the tool itself. The framework calls this method before invoking `call()` — the tool never calls it manually. If the result has `behavior: 'deny'`, the framework sends a `tool_result` with `is_error: true` back to the LLM and never invokes `call()` at all. This makes the permission boundary declarative: a tool that omits `checkPermissions` gets the default implementation, which returns `{ behavior: 'allow' }` unconditionally.

Three permission modes control how the gate behaves: `'default'` prompts the user for sensitive operations, `'bypass'` grants everything silently, and `'auto'` approves operations matching pre-approved rules.

## Why this matters

Claude Code executes arbitrary shell commands, reads and writes files, and can make network requests. Running in a developer's local environment means a mistake has real consequences — files can be deleted, credentials can be exfiltrated, production systems can be modified. The permission system is a defense-in-depth layer: even if the LLM produces a malicious or incorrect tool call, the user is prompted before the side effect happens.

The system is per-input, not per-tool. `checkPermissions` receives the full parsed input, so a bash tool can be auto-approved for `git status` (read-only) while still prompting for `rm -rf` (destructive). Rules match tool names, command patterns, and file path globs, making the policy composable and auditable.

## How to apply it

1. Implement `checkPermissions(input, context)` as a method on your tool definition — the framework calls it before `call()`, so no manual invocation is needed inside `call()`.
2. Inside `checkPermissions`, read `context.getAppState().toolPermissionContext` and pass it to the appropriate utility (e.g., `checkReadPermissionForTool`, `checkBashPermissionForTool`) from `src/utils/permissions/`.
3. When defining a new tool, implement `isDestructive?(input)` if the tool performs irreversible mutations. This signals to the permission UI that extra confirmation is appropriate.
4. To add auto-approve rules, write them to `~/.claude/settings.json` under `permissions.allow`. Rules use glob-like matchers: `Bash(git *)` approves all git bash commands; `FileRead(src/*)` approves file reads in src/.
5. The `toolPermissionContext` in AppState carries the current rules — read it via `context.getAppState().toolPermissionContext`.

## In the source

```typescript
// Source: src/Tool.ts (checkPermissions in ToolDef)
// checkPermissions is optional — buildTool provides a default that returns { behavior: 'allow' }
checkPermissions(
  input: z.infer<Input>,
  context: ToolUseContext,
): Promise<PermissionResult>

// PermissionResult behaviors:
// { behavior: 'allow'; ... }         — proceed immediately
// { behavior: 'deny'; message: string; ... } — framework rejects, call() is never invoked
// { behavior: 'ask'; ... }           — framework prompts user before proceeding
// { behavior: 'passthrough'; message: string; ... } — delegate to next layer

// Source: src/tools/FileReadTool/FileReadTool.ts (checkPermissions implementation)
async checkPermissions(input, context): Promise<PermissionDecision> {
  const appState = context.getAppState()
  return checkReadPermissionForTool(
    FileReadTool,
    input,
    appState.toolPermissionContext,
  )
},

// call() — framework has already run checkPermissions; no manual permission call needed
async call({ file_path, offset, limit }, context, _canUseTool?) {
  const content = await readFile(file_path)
  return { data: content }
},

// Source: src/utils/permissions/ (rule matching)
// Permission rules in settings.json:
// {
//   "permissions": {
//     "allow": ["Bash(git *)", "FileRead(**/*.ts)"],
//     "deny": ["Bash(rm *)"]
//   }
// }
```

The key non-obvious detail: Zod schema validation happens BEFORE `checkPermissions` is called. The framework validates the input first; `checkPermissions` is only invoked with valid, parsed arguments. This means permission rules can safely inspect typed fields (e.g., `input.command`) without needing to handle malformed input.

## Apply it to your code

**Before** — tool with no `checkPermissions` method (executes without any permission check):
```typescript
{
  // No checkPermissions — buildTool default returns { behavior: 'allow' } unconditionally
  async call({ file_path }, context, _canUseTool?) {
    // Directly executes without asking the user — dangerous for destructive ops
    await fs.unlink(file_path)
    return { type: 'success', data: `Deleted ${file_path}` }
  },

  isDestructive: (_input) => true,
  isReadOnly: (_input) => false,
}
```

**After** — same tool with `checkPermissions` delegating to the appropriate utility:
```typescript
{
  // checkPermissions is called by the framework before call() — never call it manually
  async checkPermissions(input, context): Promise<PermissionDecision> {
    const appState = context.getAppState()
    return checkWritePermissionForTool(
      DeleteFileTool,
      input,
      appState.toolPermissionContext,
    )
  },

  async call({ file_path }, context, _canUseTool?) {
    // Framework already ran checkPermissions — safe to execute here
    try {
      await fs.unlink(file_path)
      return { type: 'success', data: `Deleted ${file_path}` }
    } catch (err) {
      return { type: 'error', error: formatFileError(err, file_path) }
    }
  },

  isDestructive: (_input) => true,
  isReadOnly: (_input) => false,
  isConcurrencySafe: (_input) => false, // File deletion is not concurrent-safe
}
```

## Signals that you need this pattern

- A tool performs file writes, shell execution, or network requests without a `checkPermissions` method
- A tool relies on the default allow-all behavior from `buildTool` for genuinely destructive operations
- `isDestructive` is not implemented for tools that delete, overwrite, or execute system commands
- Users report that a tool never prompts for confirmation even in `'default'` permission mode
- A new tool is added that can access credentials or sensitive paths without any audit trail

## Signals that you're over-applying it

- Read-only tools (file reads, glob, grep) don't need `isDestructive` — `isReadOnly: true` is sufficient signal
- If permission mode is `'bypass'` (CI/headless), `checkPermissions` always returns allow — don't add extra guards on top
- Don't implement custom permission logic inside `call()` — use `checkPermissions`; custom checks create policy inconsistencies

## Works with

- `tool-definition` — where `checkPermissions` fits in the Tool interface
- `domain-model` — how `toolPermissionContext` lives in AppState
- `error-handling` — what to return when permission is denied
