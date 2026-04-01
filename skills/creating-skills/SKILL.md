---
name: creating-skills
description: |
  Use when extracting an engineering pattern from a codebase and packaging it as a skill for an AI assistant. Triggered when: converting a recurring code pattern into a SKILL.md, wanting your AI to apply a specific pattern automatically, or validating whether an existing skill is good enough to ship. Covers structure, the real-code requirement, the two-layer generic/specific design, and six quality tests.
---

# Creating Skills

## Overview

A skill is a unit of transferable understanding, not a documentation page. It teaches an AI assistant a pattern well enough that it applies the pattern correctly without being reminded — including knowing when *not* to apply it.

Every skill has two layers. The **generic layer** stands alone: someone who has never seen your codebase reads it and understands the pattern. The **source layer** grounds it: real code from a real file proves the pattern is battle-tested, not invented.

If a skill only has the source layer, it's internal documentation. If it only has the generic layer, it's a blog post. Both layers together make it a skill.

---

## Structure

Every skill follows this order:

```
---
name: skill-name
description: Use when [specific triggering conditions — NOT a summary of what the skill does]
---

# Skill Name

## The pattern          ← generic, no codebase knowledge required
## Why this matters  ← domain connection: what breaks for users without this
## How to apply it      ← concrete steps
## In the source  ← real source code with // Source: file/path.ext
## Apply it to your code  ← before/after with // WHY: comments on every change
## Signals that you need this pattern
## Signals that you're over-applying it
## Works with           ← related skills
```

---

## The real-code requirement

The pattern section must contain code extracted from the actual source — not code written to illustrate the concept.

**How to tell the difference:**
- Invented code looks clean, minimal, and perfectly illustrative
- Real code has imports, handles edge cases, and sometimes does things that seem unrelated to the pattern

Real code earns trust. Invented code loses it the moment a reader opens the actual file and finds something different.

Every code block in section 4 must have a `// Source: path/to/actual/file.ext` comment. If you can't point to a file, the code doesn't belong in the pattern section.

---

## The two-layer rule

The generic section ("The pattern") must pass the **stranger test**: someone who has never heard of the source codebase reads it and can apply the pattern to their own project.

Check: does the generic section use jargon specific to the source codebase? If yes, define it or move it to the source-specific section.

The source section ("Why this matters" + "In the source") is allowed to be specific — that's its job. But it should connect the pattern to something a user would experience, not just explain the code.

---

## Before / after

The before block shows realistic code — not a strawman. A competent developer would genuinely write the "before" without knowing the pattern. If the before looks obviously wrong, the transformation isn't teaching anything.

Every meaningful change in the after block gets a `// WHY:` comment. The comment explains the reasoning, not the action.

```typescript
// WHY: schema construction deferred — paid at first call, not at every module import
const schema = lazySchema(() => z.object({ path: z.string() }))
```

---

## The six tests

Run these before shipping a skill. A skill that fails any test needs to be rewritten, not shipped with a note.

**1. The stranger test**
Could someone who has never heard of the source codebase use the generic section to write better code in their own project? If the generic section uses unexplained jargon, it fails.

**2. The trigger test**
Does the description cause the skill to load at the right times — and stay quiet at the wrong times? The description is the only thing an AI reads to decide whether to load a skill. It must name specific situations, not summarise what the skill does.

```yaml
# Fails — summarises the skill, not the triggering situation
description: Teaches the permission system with checkPermissions and CanUseToolFn

# Passes — names when to load it
description: |
  Use when writing a capability that performs file writes, shell execution,
  or network requests, or when understanding why a tool call was denied.
```

**3. The real-code test**
Does section 4 contain actual source code with a file reference? If the code was written to illustrate the pattern, it fails.

**4. The before/after test**
Does reading the before make you feel the problem? Does reading the after make the improvement obvious? If both look the same, the transformation isn't teaching anything. Use domain-appropriate entities — not `foo`, `bar`, or `MyService`.

**5. The over-application test**
Does the skill tell you when not to use the pattern? A concrete situation where the pattern would be overkill — with the simpler alternative named. Generic advice ("don't over-engineer") fails this test.

**6. The domain connection test**
Does "Why this matters" make a claim about user-visible behaviour — not just code style? Ask: what breaks for a user if this pattern isn't followed? That's the answer this section should give.

---

## Common mistakes

**Description summarises the skill instead of naming when to load it.**
The AI reads the description to decide whether to load the skill. If the description says what the skill teaches, the AI may follow the description instead of reading the skill. Descriptions are triggering conditions only.

**Generic section uses source-codebase jargon without defining it.**
The stranger test fails. Move the jargon to the source section or define it before using it.

**Code examples are invented, not extracted.**
Fails the real-code test. Go back to the source and find a real example. Messier real code teaches more than clean invented code.

**Before block shows obviously wrong code.**
No one would write it that way. The transformation teaches nothing. Rewrite the before to show what a good developer would genuinely produce without the pattern.

**Over-application section says "don't over-engineer".**
This is not guidance. Name a specific situation where the pattern is overkill and what to do instead.

---

## Works with

- `domain-model` — understanding how capabilities, state, and context compose before writing skills about them
- `skill-and-command-dispatch` — how skills load and trigger; understanding this shapes how you write the `description` frontmatter
