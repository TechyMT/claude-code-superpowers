# Claude Code Superpowers — Repository Guide

This repository is a collection of engineering skills for AI coding assistants. Each skill is a structured markdown file that teaches a specific pattern — extracted from production source code, validated against real implementations, and tested against the six-test standard to ensure it's transferable beyond the source codebase.

---

## Repository structure

```
skills/          — one subdirectory per skill, each containing SKILL.md
docs/            — REFERENCE.md: domain model, architecture overview, vocabulary
assets/          — banner.svg, banner-footer.svg
CLAUDE.md        — working guide for Claude Code agents
AGENTS.md        — this file: working guide for Codex/OpenAI agents
GEMINI.md        — skill includes for Gemini CLI
README.md        — user-facing introduction, AI-native format
```

---

## Adding a new skill

1. Create `skills/[skill-name]/SKILL.md`
2. Follow the skill structure in `CONTRIBUTING.md` exactly
3. Add a row to the skills table in `README.md`, `CLAUDE.md`, and `AGENTS.md`
4. Add an `@` include line to `GEMINI.md`
5. Run the six-test standard (see `CONTRIBUTING.md`) before committing

**Naming:** lowercase, hyphens only, verb-first where possible (`creating-skills`, `error-handling`, not `skillCreation` or `ErrorHandling`)

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for the full skill structure, section-by-section requirements, and the six-test standard. That file is the single source of truth for contribution rules.

---

## Reference

[`docs/REFERENCE.md`](docs/REFERENCE.md) is the foundational context for this repo — domain model, architecture overview, and vocabulary. Read it before working on any skill.

---

## What not to do

- Do not add skills that only make sense if you are contributing to the source codebase
- Do not use source-codebase-specific function names or file paths in the generic sections
- Do not write code examples to illustrate a pattern — extract real code from the source
- Do not update `CLAUDE.md` or `AGENTS.md` with source-codebase-specific internal conventions

---

## Available skills

Skills are ordered from foundational to advanced. `domain-model` through `types-and-interfaces` are prerequisites for most others; read them first.

| Skill | Loads when | What it teaches |
|---|---|---|
| `domain-model` | Building any new capability, command, or feature | Six-concept model for capability-driven systems and how they compose |
| `error-handling` | Writing any capability that can fail | Throw typed errors; the framework formats them for the consumer |
| `types-and-interfaces` | Designing new types or data shapes | Schema as single source of truth; discriminated unions for multi-shape outputs |
| `tool-definition` | Defining a new typed capability | Schema-first design: one definition drives types, validation, and API docs simultaneously |
| `permission-system` | Writing a capability with side effects | Permissions declared on the capability, evaluated by the framework before execution |
| `async-concurrency` | Writing read-only or long-running capabilities | Per-call concurrency declarations, cancellation propagation, parallel I/O |
| `build-tool-factory` | Creating a new capability object | Factory pattern with safe defaults — only override what your capability actually needs |
| `module-organisation` | Deciding where new code lives | Organise by responsibility boundary, not by feature |
| `naming-conventions` | Naming files, types, functions, or variables | Names encode role: verb prefixes, boolean prefixes, callback prefixes, constant casing |
| `hot-paths` | Writing startup code or frequently-called functions | Deferred schema construction, memoised computation, parallel initialisation |
| `state-management` | Reading or writing shared state from a tool or task | Immutable state with atomic reducer updates — spread at every level, return `prev` unchanged as a no-op signal |
| `task-system` | Spawning long-running background work | Disk-backed background tasks with explicit lifecycle |
| `system-boundaries` | Integrating an external API, process, or service | Each external system gets a boundary module that owns its failure modes |
| `observability` | Adding logging or instrumentation | PII-safe structured event logging, duration tracking, telemetry vs debug distinction |
| `skill-and-command-dispatch` | Writing a slash command or skill | Prompt injection vs. local execution; inline vs. forked context isolation |
| `creating-skills` | Authoring a new SKILL.md from a codebase pattern | The two-layer structure, the real-code requirement, and the six quality tests |