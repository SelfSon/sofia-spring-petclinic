---
name: verify
description: Verify recent changes by discovering and running this project's actual build/test/lint commands (not assumed defaults), reporting raw evidence rather than narrated success, and optionally demonstrating behavior live. Use after /implement, or any time the user wants to confirm changes actually work before committing. Final step of the spec -> plan -> implement -> verify pipeline.
user-invocable: true
---

# /verify — Confirm it actually works

Arguments passed: `$ARGUMENTS` — optional scope hint (e.g. a package name,
a spec slug). If omitted, verify whatever changed in the working tree.

## Process

1. **Discover the project's real tooling before running anything.** Read
   the actual config, don't assume:
   - CI workflow files (`.github/workflows/*.yml`, `.gitlab-ci.yml`, etc.)
     if present — these are the actual bar the code has to clear, mirror
     their commands exactly rather than guessing plugin invocations.
   - Build file for what's actually *bound* to which phase/script (e.g. a
     Maven plugin `<execution>` might only run a narrow check, not its
     tool's full default ruleset — check the `<configuration>`, don't
     invoke the plugin goal bare and assume it reflects what CI runs).
   - `package.json` scripts, `Makefile` targets, or equivalent, for the
     project's own names for "test"/"lint"/"build".

2. **Run narrow before broad.** New/changed tests first for fast signal,
   then the full test suite, then a full build matching what CI would run
   (`mvn verify`, `npm run build && npm test`, etc. — whatever step 1
   found).

3. **Pull results from raw artifacts, not console narration.** Read the
   actual test report files (surefire/junit XML, coverage output, whatever
   the toolchain produces) and quote real numbers/paths/exit codes. If
   something is reported as "passed", it must be because a file or exit
   code says so, not because the run scrolled by without visible errors.

4. **If the change is user-observable** (an endpoint, a CLI command, a UI
   change), offer to actually run the app and demonstrate the behavior
   live (curl, a script, a browser) rather than resting solely on
   automated tests.

5. **Report:**
   - what was run (exact commands)
   - what passed, with the raw evidence (report path, exit code, counts)
   - what was NOT covered and why (e.g. no test for concurrent access
     under real load, only a simulated one)
   - anything the user might want to independently re-verify themselves
     (rerun locally, check in IDE, push to CI) — don't imply your run is
     the only possible confirmation

6. **Never commit from this skill.** If everything checks out, say so
   plainly and let the user decide on committing/PR separately.

## Notes

- If step 1 turns up nothing (no CI config, no explicit scripts), say so
  explicitly and fall back to the ecosystem's obvious default (`mvn test`,
  `npm test`, `pytest`, ...) rather than silently picking one.
- A clean run with zero output is not evidence of anything by itself —
  always cite what you actually read (a report, an exit code), not the
  absence of errors in a terminal scrollback.
- Keep this skill codebase-agnostic — tooling discovery in step 1 happens
  fresh every run, never hardcoded to one project's build system. That's
  what makes it safe to copy into any other repo's
  `.claude/skills/verify/`.
