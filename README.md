# ntui

A terminal user-interface library for **[Nitpick](https://github.com/alternative-intelligence-cp/nitpick)** — the
safety-critical systems language. No dependencies, no libc, no terminfo,
no C anywhere in the artifact. It talks to the kernel and to the terminal,
and to nothing else.

> **Status: planning.** No code yet. The specification set is being written in
> [`meta/specs/`](meta/specs/) and the plan in [`meta/roadmap/`](meta/roadmap/),
> in the same order and by the same discipline the compiler used — specs first,
> then a cycle map, then execution-grade subcycles, then code. The compiler
> itself is at cycle 1.5 (verification); this library is planned now so that
> implementation can start the day the language stops moving.

---

## Why another TUI library

Because the interesting parts of a terminal interface are the parts every
existing library treats as housekeeping, and Nitpick makes them checkable.

**Terminal restoration is a safety property, not a cleanup convention.**
A program that dies with the terminal in raw mode, on the alternate screen,
with the cursor hidden and mouse reporting on, leaves the user with a shell
that does not echo and does not scroll. In Nitpick `defer` deliberately does
**not** run on a trap — at trap time the state of the process is unknown — so
cleanup that matters lives in `failsafe`, the mandatory shutdown handler. `ntui`
registers its terminal state in a preallocated, allocation-free structure that
`failsafe` can walk in a degraded process, and restores the terminal with one
`write(2)` before anything else runs. That is the same mechanism the language
already uses for the allocation registry and the driver registry, applied to
the one piece of global state a terminal program owns.

**Rendering is deterministic to the byte.** The same screen state produces the
same escape sequence, every time, on every machine. Nothing is inferred from
`isatty`, nothing varies with `$TERM` at runtime once capabilities are resolved,
and nothing depends on a system database. That is what makes a rendered frame
testable: a test asserts on bytes, and a golden screen is a fact rather than an
approximation.

**Every wait has a deadline and every operation returns `Result<T>`.** There is
no unbounded read on standard input, no unchecked write, and no operation that
can hang a program without a code path that says so. Input arrives through the
language's epoll reactor as a task suspension, so a terminal that stops talking
stalls nothing else.

**Layout arithmetic cannot silently wrap.** Plain integer overflow traps in
Nitpick, and cell coordinates carry bounds the type system checks. The geometry
that decides where a character lands is the part of a TUI that is wrong most
often and noticed least, and here it is the part the compiler is best at.

**No terminfo.** The system terminfo database is a binary format read from
untrusted files, describing terminals that mostly no longer exist, and it is
routinely wrong about the ones that do. `ntui` carries a small compiled-in
capability table and corrects it at startup by **asking the terminal itself**
(DA1, XTVERSION, DECRQM). One code path, no parser, no machine state in the
output.

---

## What it will provide

Layered, with every layer usable on its own:

| Layer | Contents |
|---|---|
| **device** | `/dev/tty`, termios raw mode, window size, `signalfd` for `SIGWINCH`/`SIGTERM`/`SIGTSTP`, the `failsafe` restore registry |
| **text** | UTF-8 decoding, grapheme-cluster segmentation (UAX #29), display width (UAX #11 + emoji) from generated tables |
| **input** | an incremental decoder for legacy CSI/SS3 keys, `modifyOtherKeys`, the Kitty keyboard protocol, SGR mouse reporting, bracketed paste, focus events and terminal replies |
| **style** | 4-bit / 8-bit / 24-bit colour with deterministic degradation, attributes, coloured underlines, OSC 8 hyperlinks |
| **screen** | a cell grid with wide-character and combining-mark handling, double buffering, damage tracking, and a diffing renderer that emits the shortest correct sequence |
| **capabilities** | the built-in table plus the runtime probe, resolved once at startup |
| **layout** | rectangles and a constraint solver (fixed / percentage / ratio / min / max / fill), in integer arithmetic that traps rather than wraps |
| **runtime** | an async event loop over the reactor: input, signals, timers and application channels, with frame scheduling and focus |
| **widgets** | block, paragraph, list, table, tabs, gauge, sparkline, chart, scrollbar, tree, text input and a braille/half-block canvas |

The application model is **immediate-mode rendering with an Elm-shaped loop**:
widgets are plain values drawn into a buffer each frame, application state is
your own struct, events reach an `update` that returns commands, and `view`
draws. Nitpick has no closures, which rules out callback-wiring architectures
and makes this one the natural fit — it is also the one that stays testable
without a terminal.

---

## Layout

```
src/          # THE LIBRARY — Nitpick source only
  core/       #   bytes, growable storage, text primitives
  unicode/    #   GENERATED width and segmentation tables
  term/       #   the device: tty, termios, size, signals, restore
  input/      #   the decoder state machine
  style/      #   colour and attributes
  screen/     #   cells, buffers, the diff renderer
  caps/       #   the capability table and the runtime probe
  layout/     #   geometry and the constraint solver
  app/        #   the event loop and frame scheduling
  widget/     #   the widget set
tests/        # unit, golden-screen, rejection, conformance, and captured fixtures
examples/     # runnable demonstrations
harness/      # the Python build and test runner, until `npkg` can build a library
tools/        # generators — the Unicode tables and the capability table
meta/specs/   # the design authority
meta/roadmap/ # the plan, in numbered cycles
docs/         # user-facing documentation, written at 1.0
```

## Specification

[`meta/specs/`](meta/specs/) is the authority on behaviour, and
[`meta/DECISIONS.md`](meta/DECISIONS.md) records every settled design decision
with its reasoning — start there when something looks unusual, because it is
recorded why.

## Plan

[`meta/roadmap/ROADMAP.md`](meta/roadmap/ROADMAP.md) is the cycle map. A cycle is
a folder, a subcycle is a file inside it, and a finished cycle moves to
`meta/roadmap/done/`.

## Requirements

Linux on x86-64, the Nitpick compiler, and LLVM 20.1.2 — the same toolchain the
compiler pins. Nothing else, at build time or at run time.

## Licence

Apache 2.0. See [`LICENSE`](LICENSE).
