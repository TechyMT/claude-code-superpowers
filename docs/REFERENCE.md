# Claude Code — Reference

This document is the context that makes the skills in this repository make sense. Read it once before diving into individual skills.

---

## 1. What Claude Code Is

Claude Code is Anthropic's AI coding assistant — a CLI that bridges an LLM with a developer's local environment. It lets the AI read and write files, run shell commands, search code, spawn subagents, and integrate with external tools via MCP, all under a user-controlled permission system. The developer retains control over what the AI can do; Claude Code is the execution layer that translates LLM decisions into local actions.

---

## 2. The Domain Model

| Concept | What it is | Key invariant |
|---|---|---|
| **Tool** | A named capability with a typed input schema, a factory-built definition, and a permission check | Input validated by Zod schema before `call()` is invoked; permissions checked by framework before `call()` |
| **Message** | An immutable turn in the conversation | Never modified after creation; system messages are UI-only and stripped before API calls |
| **ToolUseContext** | Runtime environment injected into every tool call | Carries state, abort signal, callbacks — no module-level globals |
| **AppState** | The mutable global state store | `DeepImmutable<T>` — updated via `setAppState(prev => next)`, never mutated directly |
| **Command** | A user-facing slash command | Either `PromptCommand` (LLM-executed) or `LocalCommand` (local function) |
| **Task** | A background operation spawned by a tool | Output written to disk file; lifecycle: pending → running → completed / failed / killed |

---

## 3. Architecture Overview

```
  Developer prompt
        │
        ▼
  ┌─────────────────────────────────────────┐
  │  main.tsx  — React/Ink TUI              │
  │  (conversation, tool output, prompts)   │
  └────────────────┬────────────────────────┘
                   │
                   ▼
  ┌─────────────────────────────────────────┐
  │  query.ts  — conversation loop          │
  │  assemble system prompt → Claude API    │◄──┐
  │  stream response → dispatch tool calls  │   │ tool results
  └────────────────┬────────────────────────┘   │ fed back
                   │ tool call                  │
                   ▼                            │
  ┌─────────────────────────────────────────┐   │
  │  Permission system  (checkPermissions)  │   │
  │  allow / deny / ask user               │   │
  └────────────────┬────────────────────────┘   │
                   │ allow                      │
                   ▼                            │
  ┌─────────────────────────────────────────┐   │
  │  Tool registry                          │   │
  │  Built-in tools + MCP tools (same path) │   │
  └──┬──────────────────────────────────────┘   │
     │                                          │
     ├─► tool.call(args, context) ──────────────┘
     │     reads/writes AppState via context
     │     throws on error (framework formats)
     │
     └─► task spawn (optional)
           output written to disk
           lifecycle: pending → running → done/failed
```

The entry point is `main.tsx`, which mounts a React TUI using Ink. The UI renders the conversation, tool output, and permission prompts as terminal components.

The conversation loop lives in `query.ts`. On each turn it assembles the system prompt, calls the Claude API, streams the response, and dispatches tool calls as they arrive. Tool results are collected and fed back to the API in the next request. This loop continues until the model stops requesting tools.

State is managed through a Zustand store. The `AppState` type is `DeepImmutable<{...}>`, meaning all nested properties are readonly at the TypeScript level. The only legal update path is `setAppState(prev => next)` — direct mutation is a compile error. Tools access and update state via `context.getAppState()` and `context.setAppState()`, not via module-level imports.

The permission system intercepts every tool call before `call()` runs. It evaluates the tool's `checkPermissions()` method against a `toolPermissionContext` to produce a `PermissionDecision` of allow, deny, or ask. If the decision is `ask`, the user is prompted in the TUI before execution proceeds. MCP servers connect at startup and register their tools dynamically into the same tool registry, so external tools go through the same permission and execution path as built-in tools.

---

## 4. Design Philosophy

### 1. Schema is the single source of truth

Tool inputs use Zod schemas. TypeScript types are derived via `z.infer<>`. The LLM's JSON schema, the TypeScript types, and the runtime validator are all the same artifact — there is no separate type definition that can drift from the schema.

### 2. Tools throw, the framework formats

`tool.call()` throws typed errors (`ShellError`, `AbortError`, `Error`). The framework in `toolExecution.ts` catches them, calls `formatError()`, and sends a `tool_result` with `is_error: true` to the LLM. Tools never return `{ type: 'error' }` — that shape does not exist in this codebase.

### 3. Permissions are declared, not checked

`checkPermissions()` is a method on the tool definition. The framework calls it before `call()`. Tools do not call any permission function inside `call()` — that would be too late and is architecturally incorrect.

### 4. State through context, not globals

Tools access state via `context.getAppState()` and `context.setAppState()`. `ToolUseContext` is the dependency injection container for a tool call. There are no module-level globals for state.

### 5. Concurrency is declared per call

`isConcurrencySafe(input)` is evaluated with the actual input at call time, not as a fixed constant on the tool class. The scheduler uses this to determine what can run in parallel. The conservative default is `false`.

---

## 5. Vocabulary Glossary

**`buildTool`** — Factory function that takes a `ToolDef` and fills in safe defaults for any methods not explicitly provided.

**`ToolDef`** — Same shape as `Tool` but with `isEnabled`, `isConcurrencySafe`, `isReadOnly`, `isDestructive`, `checkPermissions`, `toAutoClassifierInput`, and `userFacingName` as optional. Used as the input to `buildTool`.

**`lazySchema`** — Wrapper that defers Zod schema construction until first access. Reduces module load time by avoiding eager schema compilation at import time.

**`isConcurrencySafe`** — Per-call method on a tool returning `true` if this specific invocation can run in parallel with other concurrent-safe calls. Evaluated against the actual input, not a static flag.

**`checkPermissions`** — Method on a tool definition returning a `PermissionDecision` (allow / deny / ask). Called by the framework before `call()`. If the result is `ask`, the user is prompted before execution.

**`ToolUseContext`** — Runtime environment passed to every tool call. Equivalent to a dependency injection container: carries `getAppState`, `setAppState`, `abortSignal`, and lifecycle callbacks.

**`AppState`** — Single mutable state store, typed as `DeepImmutable<T>`. Updated via `setAppState(prev => next)`. Never mutated directly.

**`DeepImmutable`** — TypeScript utility type making all nested properties `readonly`. Direct mutation is a compile error; all updates must go through `setAppState`.

**`PromptCommand`** — Slash command that injects a prompt template into the conversation for the LLM to act on. The LLM does the work; the command supplies the framing.

**`LocalCommand`** — Slash command that runs a local function directly, bypassing the LLM entirely.

**Fork** — Execution mode where a skill or command spawns an isolated subagent. The subagent runs its full conversation independently; only the final result returns to the parent conversation.

**MCP (Model Context Protocol)** — Protocol for extending Claude Code with external tool and resource servers. MCP servers register tools into the same registry as built-in tools, going through the same permission and execution pipeline.

---

## 6. Where to Start

1. **Read this document first.** The domain model table in section 2 is the frame that makes all other skills make sense.
2. **Then read the skill most relevant to your current task.** Each skill is self-contained and cross-references the concepts defined here.
3. **You do not need to read all 16 skills.** They cover independent topics; pick what applies.
