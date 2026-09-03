# `ntui` specifications

This directory is the **authority on what `ntui` does**. Code that disagrees
with a document here is a defect in the code; a document that turns out to be
wrong is amended by a decision recorded in
[`../DECISIONS.md`](../DECISIONS.md), never by editing the text and moving on.

That discipline is borrowed, deliberately, from the compiler repository, where
the cycle notes record the same finding over and over: **the compiler and the
thing that describes it have to be diffed, because reading either alone never
reveals the gap.** A specification nothing is held to is decoration.

## Reading order

Read the first three before proposing anything. They contain the constraints
that make this library different from every other TUI library, and most
proposals that look reasonable in the abstract die on one of them.

| # | Document | What it settles |
|---|---|---|
| 1 | [`SAFETY.md`](SAFETY.md) | the `failsafe` contract, the error budget, the resource discipline — **the constraints, and where they come from** |
| 2 | [`TERMINAL_MODEL.md`](TERMINAL_MODEL.md) | the device: `/dev/tty`, termios, size, signals, the restore registry, the lifecycle |
| 3 | [`BUILD.md`](BUILD.md) | how this is built and tested today, and the module and import conventions |
| 4 | [`TEXT_MODEL.md`](TEXT_MODEL.md) | UTF-8, grapheme clusters, display width, and where the tables come from |
| 5 | [`CAPABILITIES.md`](CAPABILITIES.md) | the built-in table, the runtime probe, and how a capability becomes a fact |
| 6 | [`INPUT_MODEL.md`](INPUT_MODEL.md) | the event vocabulary and the decoder, protocol by protocol |
| 7 | [`STYLE_MODEL.md`](STYLE_MODEL.md) | colour, attributes, degradation, hyperlinks |
| 8 | [`SCREEN_MODEL.md`](SCREEN_MODEL.md) | cells, buffers, damage, and the diffing renderer |
| 9 | [`LAYOUT_MODEL.md`](LAYOUT_MODEL.md) | rectangles and the constraint solver |
| 10 | [`EVENT_MODEL.md`](EVENT_MODEL.md) | the runtime: the loop, timers, commands, the application shape |
| 11 | [`WIDGET_MODEL.md`](WIDGET_MODEL.md) | the widget contract and the set |
| 12 | [`TESTING.md`](TESTING.md) | the harness, the headless device, the golden-screen oracle |
| 13 | [`VERIFICATION.md`](VERIFICATION.md) | the proof obligations this library carries into the compiler's cycle 1.5 |
| 14 | [`COMPAT.md`](COMPAT.md) | the terminal support matrix, and what is tested on what |
| 15 | [`GLOSSARY.md`](GLOSSARY.md) | the words, used one way each |

## What is normative, and what is not

- A **rule** stated in these documents is normative. Rules read as statements of
  fact about the library ("a cell holds exactly one grapheme cluster"), not as
  intentions.
- A **rationale** paragraph explains why, and carries no obligation of its own.
- A **decision reference** — `T-nnn` — points at
  [`../DECISIONS.md`](../DECISIONS.md), which holds the argument, the
  alternatives considered, and the date. `D-nnn` points at the **compiler's**
  `meta/specs/DECISIONS.md`; those are language decisions and are not ours to
  amend.
- An **open item** is listed at the end of the document that owns it, and is
  mirrored in [`../OPEN_QUESTIONS.md`](../OPEN_QUESTIONS.md) with a
  recommendation. A question that lives only in a conversation evaporates.

## The language, in one paragraph, for a reader arriving from C

Nitpick has no exceptions and no unhandled errors: every function returns
`Result<T>` except `main` and `failsafe`. There is no garbage collector; the
default regime is static ownership with destruction at scope exit, owning values
are move-only, and borrows are second class — they pass down the call stack and
never up, never across a thread spawn, never across an `await`. Plain integer
overflow **traps**. There are no closures. There is no `select` on channels.
`defer` runs on every normal exit path and **not** on a trap. Anything uncaught
routes through a mandatory `failsafe` handler, which is the last code that runs
in a process that has decided to stop. Read the compiler's
`meta/specs/` for the full statement; the pieces that bite hardest here are
enumerated in [`SAFETY.md`](SAFETY.md) §1.
