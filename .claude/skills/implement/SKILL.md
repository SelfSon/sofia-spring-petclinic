---
name: implement
description: Execute an approved implementation plan (from /plan) with task tracking, matching the codebase's existing conventions, stopping before commit. Use after a plan has been approved via ExitPlanMode. Third step of the spec -> plan -> implement -> verify pipeline.
user-invocable: true
---

# /implement — Execute an approved plan

Arguments passed: `$ARGUMENTS` — optionally a plan file path or spec slug.
If omitted, use the plan most recently approved in this session, or look
for it via the spec file's "## Plan" section.

## Process

1. **Read the plan file fully** (and the spec it points back to, for the
   "why" behind each change).

2. **Create one task per file/step** in the plan's file list via
   `TaskCreate`. Keep them granular enough that progress is visible, not
   so granular that bookkeeping outweighs the work.

3. **Before writing any NEW file**, read one existing sibling file of the
   same kind in this repo (same layer: e.g. an existing controller if
   you're adding a controller, an existing test if you're adding a test)
   and match its conventions: license/header comment, import grouping and
   ordering, naming, doc-comment density, error-handling style. Do not
   invent a new house style for one file.

4. **Work through tasks in dependency order**, marking each `in_progress`
   before starting and `completed` only once it's actually done (not
   "mostly done"). If a step turns out to need something the plan didn't
   anticipate (a renamed dependency, a missing package, a conflicting
   existing name), fix it, but flag the deviation in your final summary
   rather than silently absorbing it.

5. **After adding any new external dependency**, do an immediate minimal
   build/compile check before writing more code on top of it — don't trust
   a remembered package/artifact name or version. Concretely: run whatever
   this project's build tool uses to resolve dependencies (`mvn compile`,
   `npm install`, `go build ./...`, etc.) right after the manifest edit,
   before touching source files that depend on it. Package/artifact names
   do change between major versions — verify against what's actually
   resolvable in this repo, not memory.

6. **Do a single sanity compile/build at the end** of implementation (not
   the full test suite — that's `/verify`'s job) to catch obvious breakage
   early.

7. **Stop.** Report what was implemented, file by file, and any deviations
   from the plan. Explicitly state this has **not** been tested or
   committed yet. Suggest `/verify` as the next step. Never commit from
   this skill, even if asked generally to "finish it up" — commits happen
   only on an explicit, separate request.

## Notes

- If mid-implementation you discover the plan's approach is actually
  wrong (not just under-specified), stop and raise it — don't silently
  route around a bad plan by improvising; that's exactly the kind of
  decision that should go back through `/plan` or a direct question to the
  user.
- Keep this skill codebase-agnostic — it adapts to whatever conventions
  step 3 finds in this repo, never assumes a specific language or style.
  That's what makes it safe to copy into any other repo's
  `.claude/skills/implement/`.
