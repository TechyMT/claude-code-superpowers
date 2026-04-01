# Changelog

All notable changes to this project will be documented here.

---

## [Unreleased]

### Added
- `build-tool-factory` skill — teaches `buildTool()` factory pattern with safe defaults
- `creating-skills` skill — the two-layer structure, real-code requirement, and six quality tests (replaces `writing-skills`)
- `docs/REFERENCE.md` — domain model, architecture overview, design philosophy, and vocabulary
- `CONTRIBUTING.md` — skill authoring guide, six validation tests, PR checklist
- Multi-agent support: Cursor (`.cursor-plugin/`), Codex (`.codex/`), OpenCode (`.opencode/`), Gemini CLI (`GEMINI.md`)
- `hooks/hooks.json` expanded from 1 matcher to 11, covering all skills

### Fixed
- `error-handling` — corrected from "return error shapes" to "throw typed errors, framework formats them"
- `tool-definition` — corrected from direct `Tool<>` implementation to `buildTool()` factory pattern
- `permission-system` — corrected from `canUseTool()` in `call()` to `checkPermissions()` method on tool def
- `types-and-interfaces` — corrected `ToolResult<T>` shape; discriminated unions apply to output data, not the result wrapper
- `domain-model` — added generic framing; fixed code examples to use `buildTool()` pattern
- `observability` — replaced non-existent `logForDiagnosticsNoPII` with `logForDebugging` and `logEvent`
- `async-concurrency` — fixed before/after to use `throw` and `{ data }` returns
- `skill-and-command-dispatch` — added generic framing for non-Claude Code audiences
- All plugin descriptions — removed false "state management" claim, aligned to consistent tagline
- `CLAUDE.md` and `AGENTS.md` — removed Claude Code internal references; rewritten as repo working guides
- LICENSE year updated to 2026
- GitHub issue templates and PR template made source-agnostic

---

## [1.0.0] — 2025

### Added
- Initial 13 skills: domain-model, tool-definition, permission-system, error-handling,
  async-concurrency, module-organisation, naming-conventions, types-and-interfaces,
  hot-paths, task-system, system-boundaries, observability, skill-and-command-dispatch
- `writing-skills` — TDD approach to skill authoring and validation (later merged into `creating-skills`)
