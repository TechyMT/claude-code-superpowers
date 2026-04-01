---
name: state-management
description: |
  Use when reading or writing shared application state from a tool, task, or service.
  Triggered when: accessing state outside a React component, updating a field in a
  shared state store, writing a helper that modifies task or session state, or
  understanding why direct mutation causes stale UI or race conditions.
---

# State Management

## The pattern

Shared mutable state is wrapped in a deep-immutable type. Direct mutation is a compile error. All updates go through a single setter that accepts a pure function `(prev: State) => State`. Returning the same reference signals "nothing changed" and skips downstream work — no re-render, no diff.

The reducer function is atomic: the entire state object is replaced in one operation. Partial updates are expressed as spreads — the fields that didn't change carry forward from `prev`; only the changed fields are new values. This makes every transition explicit, traceable, and race-condition-safe.

## Why this matters

Claude Code's AppState is a large object — UI flags, permission context, task records, session configuration — shared across tools running concurrently, React components rendering in a terminal, and background task workers. Without an immutability contract, a tool updating `tasks` could race with a UI component reading `toolPermissionContext` mid-update, producing torn reads.

The `DeepImmutable<T>` wrapper makes that impossible at the type level. TypeScript rejects direct assignment to any nested field. The only way to change state is through the reducer, which replaces the entire object atomically. The reference-equality short-circuit (`return prev` when nothing changed) prevents the scheduler from scheduling re-renders for no-op updates — important when multiple tasks call `setAppState` in rapid succession.

## How to apply it

1. Read state via `context.getAppState()` — this returns the current immutable snapshot.
2. Write state via `context.setAppState(prev => next)`. Always return a new object; never mutate `prev`.
3. For nested updates (e.g. a field inside `tasks`), spread at every level: `{ ...prev, tasks: { ...prev.tasks, [id]: updated } }`.
4. When the update is conditional and the condition is false, return `prev` unchanged — this is the reference-equality no-op signal.
5. For repeated nested updates to the same sub-object, extract a helper that handles the spread and the no-op check — if the updater returns the same reference, return `prev` unchanged to avoid triggering subscribers unnecessarily.
6. Never access state from module-level variables. State always comes from `context.getAppState()`.

## In the source

```typescript
// Source: src/state/AppStateStore.ts
export type AppState = DeepImmutable<{
  verbose: boolean
  tasks: Record<string, TaskState>
  toolPermissionContext: ToolPermissionContext
  // ... ~100 more fields
}>
// DeepImmutable makes every field and nested field readonly.
// Direct mutation — state.verbose = true — is a TypeScript error.

// Source: src/utils/task/framework.ts
type SetAppState = (updater: (prev: AppState) => AppState) => void

export function updateTaskState<T extends TaskState>(
  taskId: string,
  setAppState: SetAppState,
  updater: (task: T) => T,
): void {
  setAppState(prev => {
    const task = prev.tasks?.[taskId] as T | undefined
    if (!task) return prev                 // no-op: task not found

    const updated = updater(task)
    if (updated === task) return prev      // no-op: updater returned same reference

    return {
      ...prev,
      tasks: { ...prev.tasks, [taskId]: updated },
    }
  })
}

// Source: src/tasks/stopTask.ts
setAppState(prev => {
  const prevTask = prev.tasks[taskId]
  if (!prevTask || prevTask.notified) {
    return prev   // Return same reference — no re-render triggered
  }
  return {
    ...prev,
    tasks: {
      ...prev.tasks,
      [taskId]: { ...prevTask, notified: true },
    },
  }
})
```

The `if (updated === task) return prev` check in `updateTaskState` is subtle but important. Without it, every call to `setAppState` would trigger subscribers even when nothing changed — in a system where dozens of tasks poll state every second, this would saturate the React render scheduler.

## Apply it to your code

**Before** — mutating state directly and storing it in a module-level variable:
```typescript
// Wrong: module-level state — shared across calls, causes stale reads
let taskStatuses: Record<string, string> = {}

async call(args, context) {
  taskStatuses[args.taskId] = 'running'       // mutation, no broadcast
  await doWork(args)
  taskStatuses[args.taskId] = 'completed'     // UI never sees this
  return { data: 'done' }
}
```

**After** — atomic reducer updates through context:
```typescript
async call(args, context) {
  // WHY: setAppState broadcasts to all subscribers atomically
  context.setAppState(prev => ({
    ...prev,
    tasks: {
      ...prev.tasks,
      [args.taskId]: { ...prev.tasks[args.taskId], status: 'running' },
    },
  }))

  await doWork(args)

  // WHY: return prev unchanged when condition not met — skips re-render
  context.setAppState(prev => {
    const task = prev.tasks[args.taskId]
    if (!task || task.status !== 'running') return prev
    return {
      ...prev,
      tasks: {
        ...prev.tasks,
        [args.taskId]: { ...task, status: 'completed' },
      },
    }
  })

  return { data: 'done' }
}
```

## Signals that you need this pattern

- A tool stores session state in a module-level variable or closure — it will be stale across calls
- A component reads state that was updated by a tool but still shows the old value
- Two concurrent tools updating the same field produce inconsistent results
- State updates inside `call()` aren't reflected in the UI until the next full render cycle

## Signals that you're over-applying it

- Pure computation inside a tool (local variables, intermediate results) doesn't need `setAppState` — only values that need to survive beyond the current call or be visible to other tools or the UI
- Don't call `setAppState` in a tight loop for progress updates — batch them or use `onProgress` callbacks instead
- Don't use AppState for data that belongs in a task's disk-backed output file — large outputs go to disk, not memory

## Works with

- `domain-model` — where AppState fits in the overall system model
- `task-system` — task lifecycle transitions are the most common AppState update pattern
- `async-concurrency` — concurrent tools reading and writing state safely
- `tool-definition` — how `context.getAppState()` and `context.setAppState()` are accessed from `call()`
