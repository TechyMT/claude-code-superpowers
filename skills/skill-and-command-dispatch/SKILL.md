---
name: skill-and-command-dispatch
description: |
  Teaches Claude Code's Skill and Command system: how slash commands (/commit, /plan) are defined as PromptCommand or LocalCommand, how skills are loaded from .claude/skills/ or bundled registry, and the inline vs. fork execution modes. Use this when creating new slash commands, writing SKILL.md files, or understanding how user-defined skills extend the conversation. The execution mode ('inline' vs 'fork') determines whether the skill shares or isolates the conversation history.
---

# Skill and Command Dispatch

## The pattern

Any AI agent framework needs a way to let users trigger named behaviors — commands that are text shortcuts for common workflows. The key design decision is **execution isolation**: does the command run inside the current conversation (sharing history and state), or in a separate fork (isolated context, only the result comes back)? Inline mode is simpler; fork mode is for commands that do extended multi-step research without cluttering the conversation history.

Commands come in two types: **prompt injection** (a template is added to the conversation for the LLM to act on) and **local execution** (a function runs directly, bypassing the LLM). Prompt injection suits open-ended tasks; local execution suits deterministic operations (opening a file, showing help text).

Claude Code applies this pattern with **PromptCommand** (prompt injection) and **LocalCommand** (local execution). A Command is a user-facing `/something` that appears in the Claude Code prompt. Skills are SKILL.md files dropped into `.claude/skills/` — they become PromptCommands automatically.

The critical split is **execution context**: `context: 'inline'` means the skill/command runs inside the current conversation, sharing message history and the current AppState. `context: 'fork'` means the skill spawns a subagent with an isolated copy of the conversation — the subagent runs, returns a result, and the result is injected back. Fork mode is for skills that do research or generate artifacts without polluting the conversation history.

## Why this matters

The skill system is Claude Code's extension mechanism. Users drop a SKILL.md file into `.claude/skills/` and it appears as a `/skill-name` command. No code compilation, no restart. The file's frontmatter (`name`, `description`, `model`, `allowedTools`) configures the command; the file body becomes the prompt template.

Fork mode exists because some skills (e.g., `/commit` which drafts a commit message and opens an editor) need to do multi-step work without their intermediate reasoning cluttering the user's conversation history. The subagent's conversation is ephemeral; only its output survives back to the main session.

## How to apply it

1. To create a new built-in command: add a file or directory to `src/commands/`. Export a `PromptCommand | LocalCommand | LocalJSXCommand`. Register it in `src/commands.ts`.
2. To create a skill via SKILL.md: write a markdown file with frontmatter (`name`, `description`) and body text as the prompt. Place in `.claude/skills/[name]/SKILL.md`. It auto-registers as a PromptCommand.
3. Choose `context: 'inline'` when the skill's output should become part of the ongoing conversation (e.g., answering a code question).
4. Choose `context: 'fork'` when the skill does extended work and should return a single result (e.g., generating a PR description, running tests and reporting back).
5. Use `allowedTools` frontmatter to restrict which tools a skill can invoke — this is a safety mechanism for skills that don't need write access.
6. Use `agent` frontmatter to specify which agent type runs a forked skill (e.g., `agent: general-purpose`).

## In the source

```typescript
// Source: src/types/command.ts (PromptCommand definition)
export type PromptCommand = {
  type: 'prompt'
  progressMessage: string    // Shown while the command runs
  contentLength: number      // Estimated prompt size for token budgeting
  argNames?: string[]        // Named slots in the prompt template: $ARGUMENTS
  allowedTools?: string[]    // Restrict which tools this command can use
  model?: string             // Override model for this command
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  pluginInfo?: {
    pluginManifest: PluginManifest
    repository: string
  }
  disableNonInteractive?: boolean
  hooks?: HooksSettings
  skillRoot?: string         // Directory containing the SKILL.md
  context?: 'inline' | 'fork'  // Execution isolation mode
  agent?: string             // Which agent type runs this (for fork mode)
  effort?: EffortValue       // thinking effort: low|medium|high
  paths?: string[]           // File paths to attach to the prompt
  getPromptForCommand(
    args: string,
    context: ToolUseContext,
  ): Promise<ContentBlockParam[]>  // Returns the prompt content to inject
}

// Source: src/types/command.ts (LocalCommand for non-LLM commands)
export type LocalCommand = {
  type: 'local'
  description: string
  call(
    args: string,
    context: ToolUseContext,
  ): Promise<void>  // Runs locally, returns nothing to the conversation
}

// Source: src/skills/loadSkillsDir.ts (SKILL.md → PromptCommand conversion)
// SKILL.md frontmatter example:
// ---
// name: commit
// description: Draft a commit message and open an editor
// context: fork
// agent: general-purpose
// allowedTools: [Bash, FileRead, Glob]
// ---
// [prompt body becomes the template]

// The skill loader reads frontmatter and creates a PromptCommand with:
// - type: 'prompt'
// - context: from frontmatter (default 'inline')
// - getPromptForCommand: returns the file body with $ARGUMENTS substituted
// - source: 'plugin' | 'builtin' based on location
```

The `argNames` field enables named arguments in the prompt template: a command with `argNames: ['filename']` and the user typing `/review src/main.tsx` substitutes `src/main.tsx` for `$FILENAME` in the prompt. Multiple names map positionally to space-separated user arguments.

## Apply it to your code

**Before** — complex analysis done inline, polluting conversation history:
```typescript
// User runs /analyze-deps, which makes 20 tool calls and produces intermediate output
// All of this appears in the main conversation history — confusing for the user
export const analyzeDepsCommand: PromptCommand = {
  type: 'prompt',
  context: 'inline',  // Wrong: all analysis steps visible in main conversation
  progressMessage: 'Analyzing dependencies...',
  contentLength: 500,
  async getPromptForCommand(args) {
    return [{ type: 'text', text: ANALYZE_DEPS_PROMPT }]
  },
}
```

**After** — analysis forked into an isolated subagent:
```typescript
// Fork mode: subagent does the multi-step analysis in isolation
// Only the final report is injected back into the main conversation
export const analyzeDepsCommand: PromptCommand = {
  type: 'prompt',
  context: 'fork',        // Isolated: intermediate steps don't appear in main conversation
  agent: 'general-purpose',
  progressMessage: 'Analyzing dependencies...',
  contentLength: 500,
  allowedTools: ['FileRead', 'Glob', 'Bash'],  // Read-only — no write access needed
  effort: 'medium',       // Don't use max thinking for a routine analysis
  async getPromptForCommand(args, toolContext) {
    const target = args || 'package.json'
    return [{
      type: 'text',
      text: `Analyze the dependencies in ${target}. Check for:
1. Outdated packages (compare to latest versions)
2. Security advisories
3. Unused dependencies

Return a concise markdown report with findings and recommendations.`,
    }]
  },
}
```

## Signals that you need this pattern

- A command does multi-step research and all intermediate tool calls clutter the conversation
- A slash command needs access to only a subset of tools but has access to all of them
- A skill needs to run on a different model (e.g., a lightweight haiku model for quick lookups)
- A command's prompt is hardcoded in TypeScript when it could be a SKILL.md file (easier to iterate)
- A command runs as 'inline' but produces output the user shouldn't have to scroll past in their history

## Signals that you're over-applying it

- Short, one-shot prompts don't need 'fork' mode — the overhead of spawning a subagent isn't worth it
- Commands that need to modify conversation state (e.g., add a file to the context) must run 'inline' — fork mode can't modify the parent's state
- Don't create SKILL.md files for commands that have complex TypeScript logic — the skill body is just a prompt template, not code

## Works with

- `domain-model` — where Commands fit relative to Tools and Tasks
- `async-concurrency` — how fork mode spawns an isolated async context
- `permission-system` — allowedTools enforcement within skill execution
