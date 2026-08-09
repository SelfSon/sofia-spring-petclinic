---
name: spec
description: Turn a rough one-line feature idea into a clarified, written spec file by grounding in the actual codebase and asking iterative clarifying questions. Use before any planning or coding starts, whenever the user proposes a new feature, asks "can we spec this out", or gives a vague implementation request that needs requirements clarified first. First step of the spec -> plan -> implement -> verify pipeline.
user-invocable: true
---

# /spec — Turn an idea into a written spec

Arguments passed: `$ARGUMENTS` — a one-line feature description, e.g.
`/spec local rate limiter for the search endpoint`. If empty, ask the user
for one before doing anything else.

This skill produces exactly one artifact: `specs/<slug>.md`. Nothing else
gets touched — no code, no plan file. The point is to convert one vague
sentence into a document that a `/plan` run (or a human) can act on without
re-asking the same questions.

## Process

1. **Pick a slug.** Kebab-case, 3-5 words, from the feature description
   (e.g. `owners-search-rate-limit`). If `specs/<slug>.md` already exists,
   ask the user whether to revise it in place or pick a new slug — don't
   silently overwrite someone else's spec.

2. **Ground yourself in the real repo before asking anything.** Spend a
   proportional amount of effort here — a few `Glob`/`Grep`/`Read` calls for
   a small feature, more for something that touches several subsystems.
   Look for:
   - the language/framework/build tool actually in use (read the manifest —
     `pom.xml`, `package.json`, `go.mod`, `Cargo.toml`, `pyproject.toml`,
     whichever exists — don't assume)
   - existing code that does something similar (naming, package/module
     layout, error-handling conventions)
   - existing config file(s) and their format, so any new config you
     propose lands in the pattern already in use, not a new one
   - existing test conventions (framework, file layout, naming)

   Do **not** write any code or config yet. This step exists so the
   clarifying questions in step 3 are grounded ("this repo already uses X
   for Y, want the same here or something different?") instead of generic.

3. **Ask clarifying questions in rounds, via AskUserQuestion.** 3-4
   questions per round, each with a recommended option and real
   alternatives (not open-ended blanks) informed by step 2. Keep asking
   further rounds only while a real ambiguity remains that would change
   what gets built — scope, behavior on edge cases, non-functional
   constraints (perf/security/backward-compat), config surface. Stop once:
   - the remaining choices have an obviously-safe, easily-reversible
     default (pick it, say so, move on), or
   - the user says something like "your call" / "just pick something"

   Do not ask about implementation *plan* details (file layout, exact
   class names) — that's `/plan`'s job, not this skill's.

4. **If there are genuinely different viable approaches** (e.g. write it by
   hand vs. pull in a library, sync vs. async, etc.), surface them here as
   part of the Q&A with brief pros/cons/estimate each — enough for the user
   to pick a direction. Deep implementation design still belongs to `/plan`.

5. **Write `specs/<slug>.md`** (create the `specs/` directory if missing)
   using the template below, filled in from steps 2-4.

6. **Show the user the summary and file path, and ask if it's right**
   before treating it as final — don't silently write and move on. If they
   want changes, edit the file, don't start over.

7. **Suggest the next step**: `/plan specs/<slug>.md`.

## Spec template

```markdown
# Feature: <name>

## Goal
One paragraph: what and why.

## Non-goals
What is explicitly out of scope, so it doesn't creep back in later.

## Context (found in repo)
Stack/build tool, relevant existing files/patterns, conventions discovered
in step 2 — with file paths.

## Decisions log
| Question | Options considered | Chosen | Why |
|---|---|---|---|

## Chosen approach
What will be built, plus 1-2 lines on alternatives that were considered
and rejected, and why.

## Configuration / parameters
Concrete values (limits, timeouts, flags, anything numeric or stringy).

## Acceptance criteria
How to tell it's done — behavior, not implementation.

## Estimate
```

## Notes

- This skill never enters plan mode and never writes code. If the user
  asks "just implement it" mid-conversation, finish the spec first (or
  confirm they want to skip straight past clarification), then hand off —
  don't blend phases.
- If `/spec` is invoked but the user's message already answers most of the
  likely questions, don't ask them again — extract what's already given and
  only ask about what's genuinely still open.
- Keep this skill itself codebase-agnostic. Its job is to *discover* the
  project's stack and conventions each run, never to assume a specific
  language, build tool, or prior project. That's what makes it safe to
  copy into any other repo's `.claude/skills/spec/`.
