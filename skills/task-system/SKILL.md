---
name: task-system
description: |
  Teaches Claude Code's Task system: background work spawned from tools or commands, typed by TaskType (local_bash, local_agent, remote_agent, in_process_teammate, local_workflow), tracked in AppState.tasks, with disk-backed output files and a lifecycle of pending→running→(completed|failed|killed). Use this when a tool needs to spawn long-running work without blocking the conversation loop, or when building features that monitor background operations.
---

# Task System

## The pattern

A Task is a unit of background work that runs independently of the main conversation loop. Tasks are spawned by tools or commands, tracked in `AppState.tasks` by ID, write their output to a disk file (not memory), and progress through a lifecycle: `pending → running → completed | failed | killed`.

The key design: tasks do not block the conversation. After a task is spawned, the LLM can continue the conversation, ask the user for more input, or respond to the task's output when it's done. The user can check task status with `/tasks`, kill a task with `/tasks kill`, or watch its output live.

## Why this matters

Some operations take seconds to minutes: running a test suite, compiling a large project, cloning a repository, running a long Temporal workflow. If these blocked the conversation loop, the user would be stuck waiting. The Task system decouples the spawn from the wait.

Disk-backed output (each task writes to a temp file, tracked by `outputFile` + `outputOffset`) is a deliberate choice over in-memory buffers. It means: (1) output is preserved across session restarts; (2) the output file can be `tail -f`'d externally; (3) the main process doesn't hold all output in memory; (4) partial output can be read back at any offset without re-running the task.

## How to apply it

1. To spawn a task: use the `TaskCreateTool` from the tool registry. It accepts a `type`, `description`, and type-specific configuration. The tool returns a `taskId`.
2. The task lifecycle is managed by the task runner for its type (e.g., `LocalAgentTask`, `LocalShellTask`). Don't manage it manually.
3. To read task status: use `context.getAppState().tasks[taskId]` — it returns a `TaskStateBase` with `status`, `outputFile`, `outputOffset`, and timing.
4. To kill a task: use `TaskStopTool` with the `taskId`.
5. To stream output: read from `outputFile` starting at `outputOffset` — increment the offset with each read.
6. When the task completes, set `endTime` and update `status` to `'completed'` or `'failed'` via `setAppState`.

## In the source

```typescript
// Source: src/Task.ts (task type definitions)
export type TaskType =
  | 'local_bash'        // Shell command running in background
  | 'local_agent'       // Claude subagent running locally
  | 'remote_agent'      // Agent running on CCR (Claude Code Remote)
  | 'in_process_teammate' // Swarm mode: another agent in same process
  | 'local_workflow'    // Complex multi-step workflow
  | 'monitor_mcp'       // MCP server health monitor
  | 'dream'             // Speculative execution mode

export type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed'

export type TaskStateBase = {
  id: string
  type: TaskType
  status: TaskStatus
  description: string
  toolUseId?: string         // Which tool call spawned this task
  startTime: number          // Unix ms
  endTime?: number           // Set when terminal state reached
  totalPausedMs?: number     // Time spent paused (for interruptible tasks)
  outputFile: string         // Path to disk-backed output file
  outputOffset: number       // Current read position in output file
  notified: boolean          // Whether user was OS-notified of completion
}

// Source: src/tasks/LocalShellTask/ (task lifecycle management)
export async function runLocalShellTask(
  task: LocalShellTaskState,
  context: TaskContext,
): Promise<void> {
  // Update status to running
  context.setAppState(prev => ({
    ...prev,
    tasks: {
      ...prev.tasks,
      [task.id]: { ...prev.tasks[task.id], status: 'running' },
    },
  }))

  const outputStream = fs.createWriteStream(task.outputFile, { flags: 'a' })

  try {
    await executeShell(task.command, {
      signal: context.abortController.signal,
      onData: (chunk) => {
        outputStream.write(chunk)
        // Increment offset so consumers can read new data
        context.setAppState(prev => ({
          ...prev,
          tasks: {
            ...prev.tasks,
            [task.id]: {
              ...prev.tasks[task.id],
              outputOffset: prev.tasks[task.id].outputOffset + chunk.length,
            },
          },
        }))
      },
    })

    // Terminal state: completed
    context.setAppState(prev => ({
      ...prev,
      tasks: {
        ...prev.tasks,
        [task.id]: {
          ...prev.tasks[task.id],
          status: 'completed',
          endTime: Date.now(),
        },
      },
    }))
  } catch (err) {
    // Terminal state: failed
    context.setAppState(prev => ({
      ...prev,
      tasks: { ...prev.tasks, [task.id]: { ...prev.tasks[task.id], status: 'failed', endTime: Date.now() } },
    }))
  } finally {
    outputStream.close()
  }
}
```

The `notified: boolean` field is subtle: it tracks whether the OS notification was sent when the task completed. Tasks that complete while the user is away from the terminal send an OS notification. This flag prevents sending the notification multiple times if AppState is re-rendered.

## Apply it to your code

**Before** — long-running operation blocking the conversation:
```typescript
async call(args, context, canUseTool) {
  const permission = await canUseTool(BuildTool, args, context)
  if (!permission.granted) return { type: 'error', error: permission.reason }

  // Blocks conversation for 30+ seconds — user can't interact
  const output = await runBuildProcess(args.target)
  return { type: 'success', data: output }
}
```

**After** — operation spawned as a background task:
```typescript
async call(args, context, canUseTool) {
  const permission = await canUseTool(BuildTool, args, context)
  if (!permission.granted) return { type: 'error', error: permission.reason }

  // Generate a unique task ID
  const taskId = generateId()
  const outputFile = path.join(os.tmpdir(), `build-${taskId}.log`)

  // Register task in AppState as pending
  context.setAppState(prev => ({
    ...prev,
    tasks: {
      ...prev.tasks,
      [taskId]: {
        id: taskId,
        type: 'local_bash',
        status: 'pending',
        description: `Build ${args.target}`,
        startTime: Date.now(),
        outputFile,
        outputOffset: 0,
        notified: false,
      },
    },
  }))

  // Spawn in background — don't await
  runBuildInBackground(args.target, taskId, outputFile, context).catch(console.error)

  // Return immediately — conversation continues
  return {
    type: 'success',
    data: `Build started (task ID: ${taskId}). Use /tasks to monitor progress.`,
  }
}
```

## Signals that you need this pattern

- A tool's `call()` awaits an operation that takes more than a few seconds
- Build, test, install, or compile operations block the conversation loop
- Long-running output is buffered in memory instead of streamed to a file
- The user can't interrupt a slow tool call without killing the whole process
- Task status (running/done/failed) is stored in a module-level variable instead of AppState

## Signals that you're over-applying it

- Operations that complete in under 2 seconds don't need background task tracking — the overhead isn't worth it
- Don't spawn a task for work that the LLM needs to see the result of before continuing — tasks are for fire-and-forget or monitored work
- Not every shell command needs to be a Task — `BashTool` handles short-lived commands directly

## Works with

- `domain-model` — where tasks fit in AppState and the overall system
- `async-concurrency` — AbortController for task cancellation
- `observability` — how task output is logged and streamed
