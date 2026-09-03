# CLAUDE.md

Guidance for Claude Code sessions working in this repository.

## What this is

## Before starting a session here

Check **[`../BOARD.md`](../BOARD.md)** — it says whether this repository
is claimed by a stream, and by which. **One writer per repository, always.**
[`../WORKSTREAMS.md`](../WORKSTREAMS.md) is the dependency graph and the
stream partition: what gates this repository, what this repository gates, and
what to do when a cross-stream gate is not ready yet.

`ntui` — a terminal user-interface library for **Nitpick**, the safety-critical
systems language at `../../nitpick`. **Status: planning.** No library code
exists yet. The specifications and the plan do.

## Read these first, in this order

0. **`../PLAYBOOK.md`**, if you have the sibling checkouts — the shared house
   rules for every Nitpick library, distilled from this one's planning pass:
   the language constraints that bite, the error-budget rule, the repository
   and roadmap conventions, and the state of the tooling. It is a workbench
   document and lives beside the checkouts rather than inside one, because it
   belongs to none of them.
1. **`meta/specs/SAFETY.md`** — the constraints and where they come from. Most
   proposals that look reasonable in the abstract die on one of §1's rules.
2. **`meta/specs/README.md`** — the index and the reading order for the rest.
3. **`meta/DECISIONS.md`** — every settled design decision with its reasoning.
   **Read this before proposing a change**, because it is recorded why.
4. **`meta/roadmap/ROADMAP.md`** — the cycle map; then the current cycle's
   `README.md`.
5. **`meta/OPEN_QUESTIONS.md`** — what is not settled, each with a
   recommendation.

## The rules that are not negotiable

- **The specifications are the authority** (T-002). Code that disagrees with
  `meta/specs/` is a defect in the code. A specification that turns out to be
  wrong is amended by a decision recorded in `meta/DECISIONS.md`, in the same
  commit — never by editing the text and moving on, and never by a comment.
- **A settled decision's text is never rewritten.** Supersede it with a new
  numbered decision that says why. This is the compiler's D-085/D-202 pattern.
- **Nine public error identities, and nine is a ceiling** (T-018). REACH-002
  makes every one an arm every consuming program's `failsafe` must name. A
  tenth needs a decision saying why a shutdown handler would treat it
  differently from all nine.
- **No dependencies** (T-007). Not the compiler's `src/`, not the compiler's
  `lib/`, not terminfo, not ncurses. Adding one is a decision that has to be
  argued.
- **Rendering is deterministic** (T-052). The renderer reads no environment,
  makes no syscall, consults no clock. A single `mono_now()` in it ends the
  claim and every golden test with it.
- **`ntui_restore()` must be callable from `failsafe`** (T-012, T-013) — no
  allocation, no lock, no await, idempotent. This is the library's headline
  safety property.
- **Never work around a compiler defect.** Record the reproduction, stop, and
  raise it. This is the compiler's own R6, and the reason applies here: a
  workaround buried in library code outlives the bug and is indefensible at
  verification time.

## The compiler constraints that shape everything

Full statement in `meta/specs/SAFETY.md` §1. The ones that bite hardest:

- `defer` does **not** run on a trap; `failsafe` is the only code guaranteed to
  run.
- There are **no closures**. Callbacks are values the runtime interprets.
- Owning values are **move-only**; borrows are second class and never escape.
- Plain integer overflow **traps**; indexing traps; division by zero traps.
- Every blocking operation carries a **mandatory deadline**.
- There is **no `select`** across channels.
- An `async` function can never be `never fails`, so `raw await …` is
  unspellable — use `relay`, `?|` or `?!`.

## Reserved words that read like ordinary names

`meta/specs/BUILD.md` §7 has the table. The ones this domain wants most:
`buffer`, `raw`, `move`, `fd`, `end`, `in`, `limit`, `any`, `on`, `error`,
`unit`, `thread`, `channel`, `atomic`, `is_err`, `fails`, `mod`, `as`, `with`,
`where`, `never`, `trit`, `nit`, `oflags`, `prot`, `mflags`, `fmode`.

The substitutes this library uses, so they are used consistently: `descr` for a
raw descriptor number, `sink` for a byte destination, `src` for a byte origin,
`hi` for an upper bound, `cap_set` for a capability set, `bound` for a layout
constraint, `mode_bits` for the restore mask.

Three shapes that surprise a C or Rust habit: adjacent string literals do not
concatenate; `discard(x);` takes parentheses and `defer { … }` takes no
trailing semicolon; declarations end `};` and control-flow blocks do not.

## Building and testing

**`npkg` cannot build this yet** (`meta/specs/BUILD.md` §1): it is the
compiler's own bootstrap ladder, and `[dependencies]` resolves to nothing.
`harness/run.py` is the runner until that changes (T-005). Until cycle 0.0.2
lands it, probes are run by hand — `meta/roadmap/0.0/0.0.0.md` §2 has the
command.

The compiler is at `../../nitpick`. Build it per its own `CLAUDE.md`. LLVM
20.1.2 exactly, pinned; `llvm-config --version` to check.

## Where things go

```
src/       the library, Nitpick only, layered per meta/specs/BUILD.md §6
tests/     unit, golden, rejection, conformance, probes, fixtures
harness/   the Python build and test runner, until npkg can
tools/     generators — Unicode tables, capability table, capture, fuzz
examples/  runnable demonstrations, built and run by the harness
docs/      user-facing documentation, written at cycle 1.0
meta/      specs, decisions, open questions, the roadmap, research
.internal/ gitignored scratch — never commit anything from here
```

## When you find something

- A **compiler defect**: record the reproduction, stop, raise it. Do not work
  around it.
- A **specification error**: fix the specification and record the decision, in
  the same commit as the code that revealed it.
- A **finding that is neither**: write it into the current subcycle's execution
  record. This project's execution records are load-bearing; the compiler's
  cross-cycle patterns exist only because one writer kept them.
