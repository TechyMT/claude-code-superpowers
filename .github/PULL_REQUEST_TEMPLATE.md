## What does this PR add or change?

<!-- Describe the skill added, updated, or fixed -->

## Which skill(s) does this touch?

- [ ] New skill: `skills/[name]/SKILL.md`
- [ ] Update to existing skill
- [ ] Infrastructure / docs change

## Source verification

Every code example must trace back to real source code from a production codebase.

- [ ] Section 4 code blocks include a `// Source: path/to/file.ext` comment (section 1 is generic — no source annotation required)
- [ ] No invented or paraphrased examples — extracted code only
- [ ] Before/after examples use domain-appropriate entities, not `foo`/`bar`
- [ ] The source codebase is identified (e.g. "extracted from X")

## Does the generic section stand alone?

- [ ] Someone unfamiliar with the source codebase can read "The pattern" section and apply it to their own project

## Tested on

<!-- Which AI assistants did you test the skill with? -->
- [ ] Claude Code
- [ ] Cursor
- [ ] Gemini CLI
- [ ] Other: ___

## Checklist

- [ ] Skill frontmatter: `name` (hyphens only) and `description` starting with "Use when..."
- [ ] Skill has all 8 sections in order: pattern, why [source] uses it, how to apply, pattern in [source], apply to your code, signals you need it, signals you're over-applying, works with
- [ ] One skill per PR
