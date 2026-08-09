---
name: plan
description: Turn an approved spec (from /spec) into a concrete implementation plan using the built-in Plan Mode workflow, without re-asking questions the spec already answered. Use after a specs/<slug>.md file exists and before any code is written. Second step of the spec -> plan -> implement -> verify pipeline.
user-invocable: true
---

# /plan — Spec to implementation plan

Arguments passed: `$ARGUMENTS` — path to a spec file, e.g.
`/plan specs/owners-search-rate-limit.md`. If omitted, look for the most
recently modified file under `specs/`; if there's more than one plausible
candidate, ask which.

## Process

1. **Read the spec file in full.** It already contains grounding
   (Context), decisions (Decisions log), the chosen approach, config
   values, and acceptance criteria. Treat it as settled — don't re-ask
   anything it already answers.

2. **Enter plan mode** (`EnterPlanMode`) and follow its normal workflow,
   but seed it explicitly with the spec's content so the Explore/Design
   phases build on it instead of starting cold:
   - Explore phase: only chase things the spec's Context section left
     genuinely open (e.g. it named a pattern to follow but didn't pin down
     the exact file) — don't re-explore what's already documented.
   - Design phase: produce a concrete file-by-file plan realizing the
     spec's "Chosen approach" and "Configuration" sections. If the spec
     left an implementation-level fork unresolved, resolve it here.
   - Review phase: if something in the spec turns out to be wrong or
     infeasible once you're actually looking at the code, say so
     explicitly via `AskUserQuestion` rather than silently deviating from
     the spec.

3. **Write the plan file** per the normal plan-mode format: a Context
   section (can largely point back at the spec instead of repeating it),
   the concrete file list, and a verification section describing how
   `/verify` should check this later (which commands, which behavior to
   observe).

4. **Call `ExitPlanMode`** as usual to get the plan approved.

5. **After approval**, append a short "## Plan" section to the spec file
   with the plan file's path (or a timestamp + one-line status), so the
   spec stays the single place that points at current state.

6. **Suggest the next step**: `/implement` (it will find the approved plan
   itself; no path needed unless there are multiple pending plans).

## Notes

- If no spec file exists yet for what the user is asking to plan, don't
  fabricate one — tell them to run `/spec` first, or ask if they want you
  to run it now as a preliminary step.
- This skill's whole value-add over plain plan mode is *not re-deriving
  what's already decided*. If you catch yourself asking a question the
  spec already has a Decisions-log entry for, stop and reread the spec.
- Keep this skill codebase-agnostic — all project-specific reasoning
  (stack, patterns, file locations) comes from the spec and from what
  plan mode's own Explore phase finds this run, never from assumptions
  baked into this skill. That's what makes it safe to copy into any other
  repo's `.claude/skills/plan/`.
