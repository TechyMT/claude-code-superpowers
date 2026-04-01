---
name: system-boundaries
description: |
  Teaches what Claude Code owns vs. what it delegates: the Claude API for LLM inference, MCP servers for dynamic tool/resource extension, the file system via sandboxed paths, the shell via BashTool with permission checking, and Git via execFile wrappers. Use this when adding integrations or deciding where a new capability should live — whether inside the tool system or in a new MCP server.
---

# System Boundaries

## The pattern

Claude Code has a clear inside/outside line. Inside: the conversation loop, tool execution, state management, permission checking, and UI rendering. Outside: the Claude API (LLM inference), MCP servers (dynamic tool extension), the local file system, the shell, Git, OAuth providers, and remote CCR infrastructure.

The boundary is managed by: the `services/api/` module for the Claude API, the `services/mcp/` module for MCP clients, sandboxed path resolution for file system access, and `execFile` wrappers (never `exec` with shell interpolation) for shell and Git commands.

## Why this matters

Each external system has its own failure modes. The Claude API can return 429 (rate limit), timeout, or return malformed tool call JSON. MCP servers can disconnect or return unexpected schemas. The file system can have permission errors or ENOENT. Shell commands can hang or have non-zero exit codes. Git can be absent or on an unexpected version.

By routing all external access through boundary modules (`services/api/`, `services/mcp/`, `utils/git.ts`), error handling and retry logic lives in one place. Tools don't need to handle rate limits or reconnection — they call the boundary, and the boundary handles the failure modes of that external system.

## How to apply it

1. **Claude API**: don't call the Anthropic API directly from a tool. Use the services in `services/api/`. This gives you automatic retry, token budget tracking, and streaming.
2. **MCP**: don't implement a custom MCP client. Use `services/mcp/MCPServerConnection`. Register tools via `mcpClients` in `ToolUseContext.options`.
3. **File system**: always resolve paths through `getResolvedPath()` or equivalent — this enforces the sandbox (additional working directories) and prevents path traversal. Use `fs.promises` (not sync variants).
4. **Shell commands**: use `execFileNoThrow()` or `execFileSafe()` from `utils/bash/` — these pass arguments as arrays (not strings), preventing shell injection. Never use `exec()` with string interpolation.
5. **Git**: use `getGitStatus()`, `getBranch()`, `getDefaultBranch()` from `utils/git.ts` — these wrap git in `execFileNoThrow` and handle the "not a git repo" case gracefully.
6. **New external service**: create a `services/[name]/` module, implement error handling for that system's failure modes, and expose a clean interface that tools can call without knowing about HTTP status codes or connection state.

## In the source

```typescript
// Source: src/utils/git.ts (Git as a boundary — never raw shell strings)
import { execFileNoThrow } from './bash/execFile.js'

// git commands are called with argument arrays, NEVER string interpolation
// This prevents injection: gitExe() ['log', '--oneline', userInput] is safe
// exec(`git log ${userInput}`) would be injectable
export const getBranch = async (): Promise<string> => {
  const result = await execFileNoThrow(
    gitExe(),
    ['--no-optional-locks', 'rev-parse', '--abbrev-ref', 'HEAD'],
  )
  return result.trim()
}

// Source: src/context.ts (graceful "not a git repo" handling at boundary)
export const getIsGit = async (): Promise<boolean> => {
  const result = await execFileNoThrow(
    gitExe(),
    ['rev-parse', '--is-inside-work-tree'],
  )
  return result.trim() === 'true'
}

// Source: src/services/mcp/ (MCP as dynamic extension, not hardcoded tools)
// MCP servers register tools at runtime — Claude Code doesn't know their schemas at compile time
// The MCPTool wrapper handles this: it defers tool lookup until ToolSearch is called
// context.options.mcpClients contains live connections to registered MCP servers

// Source: src/tools/BashTool/BashTool.tsx (sandbox enforcement)
async call(args, context, canUseTool) {
  // Path access is not checked inside BashTool — the permission system handles it
  // But file-access tools check against additionalWorkingDirectories in AppState:
  const resolvedPath = getResolvedPath(args.file_path, {
    additionalWorkingDirectories: context.getAppState().settings.additionalWorkingDirectories,
    cwd: getCwd(),
  })
  // If outside sandbox: permission system will deny before execution
}
```

The key security detail: Claude Code uses `execFile()` (argument arrays) everywhere, never `exec()` (shell strings). This applies to git, bash helpers, and any other subprocess. A tool input of `"$(cat ~/.ssh/id_rsa)"` would be a literal string argument to the subprocess, not executed by the shell.

## Apply it to your code

**Before** — tool calling external system without boundary module:
```typescript
async call(args, context, canUseTool) {
  // Wrong: direct fetch to external API with no retry, no rate limit handling
  const response = await fetch(`https://api.github.com/repos/${args.repo}`)
  const data = await response.json()
  return { type: 'success', data }
}
```

**After** — tool using a boundary module:
```typescript
// src/services/github/index.ts — boundary module owns HTTP concerns
export async function getRepoInfo(repo: string, signal?: AbortSignal): Promise<RepoInfo> {
  const response = await fetchWithRetry(`https://api.github.com/repos/${repo}`, {
    headers: { Authorization: `Bearer ${await getGithubToken()}` },
    signal,
  })
  if (response.status === 404) throw new GitHubNotFoundError(repo)
  if (response.status === 403) throw new GitHubRateLimitError()
  if (!response.ok) throw new GitHubAPIError(response.status, response.statusText)
  return response.json() as Promise<RepoInfo>
}

// Tool calls boundary, not raw fetch
async call(args, context, canUseTool) {
  const permission = await canUseTool(GitHubTool, args, context)
  if (!permission.granted) return { type: 'error', error: permission.reason }

  try {
    const info = await getRepoInfo(args.repo, context.abortController.signal)
    return { type: 'success', data: info }
  } catch (err) {
    if (err instanceof GitHubNotFoundError) {
      return { type: 'error', error: `Repository ${args.repo} not found.` }
    }
    if (err instanceof GitHubRateLimitError) {
      return { type: 'error', error: 'GitHub API rate limit exceeded. Try again in a minute.' }
    }
    return { type: 'error', error: `GitHub API error: ${err instanceof Error ? err.message : String(err)}` }
  }
}
```

## Signals that you need this pattern

- A tool directly calls `fetch()`, `exec()`, or reads from `process.env` without going through a service module
- Git commands are constructed with string interpolation (`\`git ${userInput}\``) instead of argument arrays
- File path resolution doesn't check `additionalWorkingDirectories` — only checks `process.cwd()`
- Rate limit and retry logic is duplicated across multiple tools instead of centralized in a service
- A tool imports from `services/api/` directly for a non-Claude-API purpose

## Signals that you're over-applying it

- Simple wrapper functions around a single stdlib call don't need a full boundary service module — a thin util function in `utils/` is fine
- Don't create a service module for external systems that are only called from one place; move the code to the tool until a second caller appears

## Works with

- `error-handling` — boundary modules define the custom error types that tools catch
- `permission-system` — the permission system gates access at the boundary
- `async-concurrency` — boundary modules accept and forward AbortController signals
