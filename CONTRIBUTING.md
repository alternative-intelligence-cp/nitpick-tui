# Contributing

`ntui` is planned before it is written, and the plan is in the repository. That
is unusual and it is deliberate: the specifications catch design mistakes that
would otherwise be found by writing the wrong code twice.

## Before you write anything

Read, in this order:

1. `meta/specs/SAFETY.md` — the constraints
2. `meta/specs/README.md` — the index and the reading order
3. `meta/DECISIONS.md` — why things are the way they are
4. `meta/roadmap/ROADMAP.md`, then the current cycle's `README.md`

## The shape of a change

**Every change belongs to a subcycle.** The current cycle's `README.md` has the
checklist; a change that is not on it either goes on it or is a finding to be
recorded first.

**A specification change is a decision.** If your change requires the library to
behave differently from what `meta/specs/` says, the specification is amended
and a numbered decision recorded in `meta/DECISIONS.md`, **in the same commit**.
A settled decision's text is never rewritten — supersede it with a new one.

**Every change is green under the full harness.** `--only` is for iterating and
never for concluding; nothing is committed on the strength of a filtered run.

## The three things that will surprise you

1. **Every public `error:` this library declares becomes a mandatory `pick` arm
   in every consuming program's `failsafe`.** The language enforces it and
   forgetting one is a compile error. The budget is nine, it is a ceiling, and
   adding a tenth is a major version. If a failure needs a distinction the
   caller cares about, it rides as a field on the returned value.

2. **The renderer is a pure function and must stay one.** No environment, no
   syscall, no clock. Everything the renderer needs is a parameter. This is what
   makes the golden suite possible, and the golden suite is the strongest thing
   this library can say about its own correctness.

3. **Terminal restoration runs on the trap path.** `ntui_restore()` allocates
   nothing, locks nothing, awaits nothing, and is idempotent, because it is
   called from `failsafe` in a process whose heap may be corrupt. A change that
   makes it allocate is a change that breaks the library's headline safety
   property.

## Tests

- **Expectations live in the test file**, as markers, and assert on codes and
  exit codes — never on message text.
- **A negative test with no expectation is a failing test.**
- **Unexpected diagnostics fail a test as surely as missing ones.**
- **A golden is re-recorded deliberately**, never as part of an ordinary run,
  and a re-record is reviewed as a diff of screen dumps.
- **Anything with a timing dimension runs forty times**, not once.
- **A red under stress is a stop sign, never a retry.**

## Compiler defects

You will find them; the library is written against a compiler that is itself
under construction. **Record the reproduction, stop, and raise it in the
compiler repository.** Do not work around it in library code: a workaround
buried here outlives the bug, is never removed, and is indefensible at
verification time.

## Style

Match the surrounding code. Public names carry their module's short prefix;
types are PascalCase; constants are SCREAMING_SNAKE. `meta/specs/BUILD.md` §7
lists the reserved words that read like ordinary names and the substitutes this
library uses instead — use those, so the tree is consistent.
