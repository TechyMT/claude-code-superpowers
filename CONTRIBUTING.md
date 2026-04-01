# Contributing to Claude Code Superpowers

## Adding a new skill

### Directory and naming

Create `skills/[skill-name]/SKILL.md`. The skill name must be:

- Lowercase with hyphens only (no underscores, no spaces)
- Verb-first: describes what the pattern does, not what it is (e.g. `build-tool-factory`, not `tool-builder`)

### Skill structure (the two-layer format)

Every SKILL.md uses the two-layer format: a generic principle layer (sections 1, 3, 5, 6, 7) that works in any codebase, over a source-grounded layer (section 4) that anchors the principle in real code. The generic sections must never reference the source codebase. The source section must never contain invented examples.

Every SKILL.md must contain exactly eight sections, in this order:

1. **The pattern** — a concise, standalone description of the technique
2. **Why this matters** — what breaks for a user if this pattern isn't followed
3. **How to apply it** — generic, step-by-step guidance for any codebase
4. **In the source** — annotated code excerpts from the source project
5. **Apply it to your code** — a before/after example using realistic, non-source code
6. **Signals you need it** — concrete triggering conditions that indicate the pattern applies
7. **Signals you're over-applying it** — a specific situation where the pattern is overkill
8. **Works with** — links to complementary skills

### Code block requirements

Every code block in **section 4** ("In the source") must include a `// Source: path/to/file.ext` comment on the first line, pointing to a real file in the source codebase. Invented examples are not permitted.

### Validation

Run all six validation tests before opening a PR (see [The six validation tests](#the-six-validation-tests) below).

### README and config updates

After the skill passes all tests, add a row to the skills table in:

- `README.md`
- `CLAUDE.md`
- `AGENTS.md`

And add an `@` include line to `GEMINI.md`:

```
@./skills/[skill-name]/SKILL.md
```

---

## The six-test standard

Run each test against your SKILL.md before opening a PR. A failure in any test requires a revision.

### 1. Stranger test

Read the generic sections (1, 3, 6, 7) aloud as if you have never seen the source codebase. If any sentence requires knowledge of the source project to make sense, rewrite it. The generic sections must stand alone.

### 2. Trigger test

Read section 6 ("Signals you need it"). Each bullet must name a concrete triggering condition — a situation the reader finds themselves in. If any bullet summarises what the skill does instead of when to reach for it, rewrite it.

### 3. Real code test

For every code block in section 4, confirm the `// Source:` path resolves to a real file in the repository. Open the file and verify the excerpt is accurate. Remove or replace any block that cannot be traced to a real source file.

### 4. Before/after test

In section 5 ("Apply it to your code"), the before snippet must depict a realistic problem that a reader would actually write. The after snippet must make the improvement obvious, with `// WHY:` inline comments explaining each change. If the before feels contrived or the after requires mental effort to understand, revise both.

### 5. Over-application test

Read section 7 ("Signals you're over-applying it"). It must name at least one specific situation — a real scenario with context — where reaching for this pattern adds unnecessary complexity. A generic warning such as "don't overuse this" does not pass.

### 6. Domain connection test

Read section 2 ("Why this matters") and section 3 ("How to apply it"). Each must explain the pattern in terms of user-visible behaviour or outcome, not just internal code structure. If a reader cannot connect the technique to something a user would experience, add that connection.

---

## Fixing an existing skill

- If a code example is wrong: locate the correct excerpt in the source codebase, update the block, and correct the `// Source:` reference.
- If a description misleads: identify which triggering condition is inaccurate and rewrite section 6 to match actual usage.
- Open a PR with the corrected SKILL.md. Include in the PR description: the original text, what was wrong, the corrected text, and which source file confirms the fix.
- Re-run the affected validation tests (at minimum: Real code test and Trigger test) and note the results in the PR.

---

## What not to contribute

- Skills that only apply to contributors of the source codebase (internal tooling, release scripts, etc.)
- Skills where section 4 ("In the source") contains invented examples — every block in that section must have a `// Source:` reference to a real file
- Skills where section 6 summarises the skill rather than naming when to load it
- Internal codebase file paths, private function names, or proprietary identifiers in the generic sections (sections 1, 3, 5, 6, 7, 8)

---

## PR checklist

- [ ] Skill passes all six validation tests
- [ ] Section 4 ("In the source") code blocks have `// Source:` references — section 1 is generic and requires no annotation
- [ ] Skills table updated in README.md, CLAUDE.md, and AGENTS.md
- [ ] `@` include line added to GEMINI.md
