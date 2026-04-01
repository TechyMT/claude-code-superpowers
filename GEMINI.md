# Claude Code Superpowers — Gemini CLI

Gemini CLI loads skills once at session start by reading this file — every `@` line below is
inlined into your context before your first prompt. This differs from trigger-based agents
(Claude Code, Cursor) where skills load on demand as your prompt matches their description
field. The trade-off: Gemini always has the full context but uses more tokens per session;
trigger-based agents are leaner but require a matching prompt to activate a skill.

Remove `@` lines for skills you don't need to reduce context size. A good starting set is
`domain-model`, `error-handling`, and whichever skill matches your current task.

# Each line below loads a skill into your Gemini CLI context. Remove lines for skills you don't need.

@./skills/domain-model/SKILL.md
@./skills/error-handling/SKILL.md
@./skills/types-and-interfaces/SKILL.md
@./skills/tool-definition/SKILL.md
@./skills/permission-system/SKILL.md
@./skills/async-concurrency/SKILL.md
@./skills/build-tool-factory/SKILL.md
@./skills/module-organisation/SKILL.md
@./skills/naming-conventions/SKILL.md
@./skills/hot-paths/SKILL.md
@./skills/state-management/SKILL.md
@./skills/task-system/SKILL.md
@./skills/system-boundaries/SKILL.md
@./skills/observability/SKILL.md
@./skills/skill-and-command-dispatch/SKILL.md
@./skills/creating-skills/SKILL.md
