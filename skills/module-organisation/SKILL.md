---
name: module-organisation
description: |
  Teaches Claude Code's module organization: tools/ for capabilities, commands/ for user-facing slash commands, services/ for business logic, state/ for AppState, utils/ for pure utilities, hooks/ for React state, and components/ for TUI rendering. Use this when deciding where to put new code or when tracing an unfamiliar behavior to its source. The division is by responsibility boundary, not by feature.
---

# Module Organisation

## The pattern

Claude Code organizes code by responsibility, not by feature. Each top-level directory under `src/` owns a specific layer of the system. Tools live in `tools/`, each in its own subdirectory named `[Name]Tool/`. Commands live in `commands/`, each in its own subdirectory or file. Business logic lives in `services/`. AppState lives in `state/`. React hooks live in `hooks/`. React components (TUI rendering) live in `components/`. Pure utility functions live in `utils/`.

The rule: a tool knows how to execute a capability. A service knows how to coordinate across multiple tools or external systems. A component knows how to render state. A hook knows how to subscribe to state. These never cross — a tool does not render, a component does not call a bash command.

## Why this matters

Claude Code is a React application running in a terminal. It has the full complexity of a web app (UI state, async data, event handling) plus the additional complexity of a capability system (tools, permissions, streaming). Without strict layer separation, service logic leaks into components, rendering logic leaks into tools, and state management becomes implicit.

The per-tool subdirectory pattern (`tools/BashTool/BashTool.tsx`) is intentional: each tool is large enough to warrant its own namespace. `BashTool.tsx` uses `.tsx` because bash output can be rendered as JSX in the TUI. `FileReadTool.ts` uses `.ts` because its output is plain data. The file extension signals whether the tool has UI concerns.

## How to apply it

1. **New capability** (file operation, shell command, web request, subagent): create `src/tools/[Name]Tool/[Name]Tool.ts[x]`. Export the tool as a named constant. Register it in `src/tools.ts`.
2. **New user command** (`/something`): create `src/commands/something/index.ts` or `src/commands/something.ts`. Register it in `src/commands.ts`.
3. **New external service integration** (API client, MCP adapter): create `src/services/[name]/`. Keep all HTTP/socket code here.
4. **New AppState field**: add it to `AppStateStore.ts`'s `AppState` type. Add it to `initialState`. Expose a selector in `state/selectors.ts`.
5. **New React hook** (subscribes to state, calls an API): create `src/hooks/use[Name].ts`.
6. **New UI component** (renders in the terminal): create `src/components/[Name].tsx`.
7. **Pure utility** (string manipulation, path normalization, type guards): create `src/utils/[name].ts`. No side effects, no imports from `state/` or `tools/`.

## In the source

```
src/
├── tools/                    # Capabilities: each is a Tool<Input, Output>
│   ├── BashTool/
│   │   ├── BashTool.tsx      # .tsx: renders bash output as JSX
│   │   ├── bashPermissions.ts
│   │   └── bashUtils.ts
│   ├── FileReadTool/
│   │   └── FileReadTool.ts   # .ts: plain data, no rendering
│   ├── AgentTool/
│   │   └── AgentTool.tsx     # Spawns subagents — has UI progress
│   └── GlobTool/
│       └── GlobTool.ts
│
├── commands/                 # User slash commands: /commit, /add-dir, /plan
│   ├── commit.ts             # Simple command: single file
│   ├── config/               # Complex command: subdirectory
│   │   ├── index.ts
│   │   └── configHelpers.ts
│   └── add-dir/
│       └── index.ts
│
├── services/                 # Cross-cutting business logic
│   ├── api/                  # Claude API client
│   ├── mcp/                  # Model Context Protocol
│   ├── compact/              # Message compaction algorithm
│   ├── tools/                # Tool orchestration (not tool implementations)
│   │   └── toolOrchestration.ts
│   └── analytics/
│
├── state/                    # AppState: single source of truth
│   ├── AppStateStore.ts      # Type + initial state (700 lines)
│   ├── AppState.tsx          # React provider
│   ├── store.ts              # Zustand factory
│   └── selectors.ts
│
├── utils/                    # Pure functions, no side effects
│   ├── messages.ts           # Message normalization pipeline
│   ├── permissions/          # Permission rule matching
│   ├── bash/                 # Bash command parsing
│   └── file.ts               # File path helpers, isENOENT()
│
├── hooks/                    # React hooks (state subscriptions)
│   ├── useCanUseTool.ts
│   └── useSettingsChange.ts
│
└── components/               # TUI components (Ink/React)
    ├── REPL.tsx
    └── Spinner.tsx
```

The `services/tools/` directory is distinct from `tools/` — it contains the orchestration logic (scheduling, concurrency, budgeting) that runs tools, not the tool implementations themselves. This prevents tools from knowing about their own scheduling.

## Apply it to your code

**Before** — new capability added directly to the query loop:
```typescript
// src/query.ts — wrong: capability logic polluting the message loop
async function handleToolCall(toolName: string, input: unknown) {
  if (toolName === 'search_logs') {
    // 50 lines of log-searching logic inline here
    const logs = await readLogsFromDisk(input.path)
    return { content: formatLogs(logs) }
  }
  // ...
}
```

**After** — capability in the correct location:
```typescript
// src/tools/SearchLogsTool/SearchLogsTool.ts
export const SearchLogsTool: Tool<typeof inputSchema, string> = {
  inputSchema,
  isConcurrencySafe: (_input) => true,
  isReadOnly: (_input) => true,
  async call(args, context, canUseTool) { /* ... */ },
  async description(input) { return `Search logs in ${input.path}` },
}

// src/tools.ts — register it
import { SearchLogsTool } from './tools/SearchLogsTool/SearchLogsTool.js'
export const getTools = (): Tools => [
  // ...existing tools...
  SearchLogsTool,
]
```

## Signals that you need this pattern

- Business logic (HTTP calls, file parsing, permission checks) exists directly in a React component
- A tool imports from `state/AppStateStore.ts` directly instead of receiving `context`
- A utility function in `utils/` has side effects or imports from `tools/`
- Commands that need access to multiple tools wire them up themselves instead of using the tool registry
- A new feature adds a file to `src/` root instead of the correct subdirectory

## Signals that you're over-applying it

- A single-function utility doesn't need its own subdirectory — a file in `utils/` is fine
- If a tool helper is used only by that tool, keep it in the same subdirectory; don't move it to `utils/`
- Not every tool needs a `.tsx` extension — only use it if the tool's output is rendered as JSX

## Works with

- `domain-model` — explains the six concepts that map to these directories
- `tool-definition` — the Tool interface all tools in `tools/` implement
- `naming-conventions` — file and export naming within each module
