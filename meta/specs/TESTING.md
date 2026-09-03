# Testing

The instruments. The compiler project's recurring finding is that **the checks
that diff two lists found the holes and the tests did not**, and that a suite
which only ever agrees with what it is handed reports green while checking
nothing. This document is `ntui`'s answer to both.

---

## 1. The stages

[`BUILD.md`](BUILD.md) §3 lists them; this is what each is *for*.

| Stage | Answers |
|---|---|
| `parse` | every source in the tree is readable by the real parser — the grammar is never quietly made partial |
| `accept` | the public API compiles in a program that only imports it |
| `check` | every documented refusal actually refuses, with exactly its code |
| `program` | the library does what it says, judged by exit code, at -O0 and under `opt -O2` |
| `golden` | the library emits exactly the bytes it is supposed to, and those bytes produce exactly the screen they are supposed to |
| `device` | the parts that need a real terminal, skipped loudly when there is none |

---

## 2. The headless device

**Rule V-1 (T-090).** `ntui_headless(rows, cols, caps)` returns a `Terminal`
whose output is a `Bytes` the test reads and whose input is a byte source the
test writes. **Everything above `src/term/` is exercised with no terminal at
all** — the input decoder, the renderer, every widget, the layout engine, and
the whole runtime loop.

This is not a convenience. It is the reason `src/term/` is a module boundary
rather than a set of calls scattered through the library, and it is what makes
the suite runnable in CI, under a debugger, and forty times over for a stress
run.

**Rule V-2.** The device layer itself — `/dev/tty`, termios, `signalfd`,
`TIOCGWINSZ` — is the residue that needs a real terminal, and it is the
`device` stage. It is small on purpose.

**Rule V-3 — a skipped device stage says so, loudly, on its own line, with the
reason.** A suite that silently skips is a suite that is green on a machine
where it ran nothing.

---

## 3. What the harness checks about the tree

Not tests. Checks that diff the library against the thing that describes it,
run on every full invocation, in the compiler's tradition where every one of
them found something on its first run.

| Check | Diffs |
|---|---|
| `check_tables_regenerate` | the committed Unicode and capability tables against what the generator emits now — a hand-edited generated file is caught |
| `check_table_invariants` | every range table is sorted, disjoint, and `lo <= hi` |
| `check_error_budget` | the count and names of public `error:` declarations against `SAFETY.md` §3's table — the budget is enforced, not remembered |
| `check_failsafe_arms` | the generated per-module arm list against a program that imports each module and compiles its `failsafe` |
| `check_layering` | every `use` edge against `BUILD.md` §6's diagram — a layering violation is caught the day it is written |
| `check_no_owning_cells` | `Cell`, `Style`, `Event` and every value stored in a fixed array declare no owning field |
| `check_constants_named` | no numeric escape-sequence literal outside `src/screen/seq.npk`, no bound outside `src/core/limits.npk` |
| `check_sequences_documented` | every escape sequence the library can emit appears in a spec table |
| `check_specs_current` | reports, does not fail: spec sections whose cited line numbers or symbol names no longer resolve |

**Rule V-4.** `check_error_budget` and `check_failsafe_arms` are the two that
matter most to a consumer, because the thing they protect — "importing this
library costs you exactly these `failsafe` arms" — is a promise no other
library in any language has to make and no compiler will check for us.

---

## 4. The input suite

Three layers, from `INPUT_MODEL.md` §14.

**Table-driven.** Every sequence in `INPUT_MODEL.md` §§4–8 with its expected
event. Each case runs twice: all bytes in one `input_feed`, and **one byte per
`input_feed`**. The second is the one that finds the partial-sequence bugs, and
it is why the decoder's signature takes a slice rather than a stream.

**Captured fixtures.** `tools/capture.npk` puts the terminal in raw mode, reads
until a sentinel, and writes a fixture file naming the terminal, the key or
gesture, and the bytes. `tests/fixtures/input/<terminal>/<gesture>.txt` is
committed. A compatibility claim in [`COMPAT.md`](COMPAT.md) means *there is a
fixture directory for that terminal and its cases pass* — a claim with a file
behind it rather than a claim.

**Fuzzing.** `tools/fuzz_input.py` drives the decoder with random bytes and
with structured mutations of the fixtures, asserting: it never traps, it always
returns to Ground within the pending bound, it consumes every byte exactly
once, and its allocation is bounded by the paste buffer. A fuzz corpus that
found something is committed as a fixture, permanently.

---

## 5. The golden oracle

**Rule V-5 (T-091) — the round trip is the test.**

```
   a Buffer the test built
        │  screen_render
        ▼
   bytes  ───────────►  the committed golden byte file
        │  mini-VT
        ▼
   a screen  ─────────►  the committed golden screen file
        │
        └──── compared against the Buffer we started from
```

A renderer bug that produces plausible-looking bytes — a cursor position off by
one, a missing style reset, a wide character whose tail was emitted, an SGR
that turned off dim along with bold — passes a byte-equality test the day the
golden was recorded and fails the round trip immediately. **This is the single
most valuable instrument in the suite** and it is why cycle 0.8 exists.

**Rule V-6 — the mini-VT is written in Nitpick, in `tests/vt/`, and is not part
of the library.** It parses the subset `ntui` can emit — CSI cursor motion,
ED/EL, SGR, DEC private modes, OSC 8, UTF-8 text — into a cell grid, and it
**fails loudly on anything it does not recognise**. A renderer that starts
emitting a new sequence has to teach the oracle about it, which is the friction
that keeps `SCREEN_MODEL.md`'s emitted-sequence table honest.

**Rule V-7 — three artifacts per golden case**, all committed: the input
description (a small program), the expected bytes, and the expected screen
rendered as text with a style legend. A reviewer reads the screen file in a
diff and sees what changed.

**Rule V-8 — regenerating a golden is a deliberate act**, `harness/run.py
--record-golden`, and the harness refuses to do it as part of an ordinary run.
A verdict that moves is a red run, never a rebaseline — the compiler's D-040
rule, applied to rendering.

**Rule V-9 — every widget has at least one golden case at three widths**
(narrow enough to clip, exact, and wide enough to leave slack) and one
containing a wide CJK character, an emoji with VS16, and a combining mark.
Those three are the shear cases, and a widget that has not been rendered
against them has not been tested.

---

## 6. Replay

**Rule V-10.** The runtime is deterministic given its inputs
(`EVENT_MODEL.md` §7), so a session is a file of
`(wake_time_ns, source, bytes)` records. `harness/replay.py` feeds one to a
headless run and compares every emitted frame against a recorded transcript.

Two uses: a bug report becomes a test (the reporter runs with
`NTUI_RECORD=path`), and a refactor of the loop is proven not to change
behaviour on a corpus of real sessions.

---

## 7. Stress

**Rule V-11.** `// stress: 40` on anything with a timing or concurrency
dimension — the escape timeout, the post channel, the signal path, the
suspend/resume cycle. The compiler found two serious defects that hid behind
single green runs and neither reproduced in fewer than about twenty.

---

## 8. Performance

**Rule V-12.** Performance is measured, recorded, and regressed against —
`harness/bench.py` writes a line per benchmark into `meta/bench/<date>.txt` and
the harness fails on a regression worse than 20% against the committed
baseline. The benchmarks are: a full-screen repaint, a single-cell change, a
scroll of a 10 000-row list, the input decoder over a 1 MiB paste, and the
layout solver over a twelve-way split.

**Rule V-13.** A benchmark is not a test and never gates on absolute numbers,
which vary by machine. It gates on the ratio to the committed baseline on the
same machine, and the baseline is re-recorded by a deliberate act like a
golden.

---

## 9. The harness is tested

**Rule V-14 (the self-check).** `harness/selfcheck.py` feeds the harness wrong
expectations and requires it to report every one as a failure:

- a `program` case with the wrong `expect-exit`;
- a `check` case expecting a code the compiler does not report;
- a `check` case that reports a code no expectation names (the D-237 rule);
- a `golden` case whose bytes differ by one byte;
- a `golden` case whose bytes match but whose *screen* differs — the case that
  proves the round trip is doing work;
- a `parse` case that does not parse;
- a table whose generator output differs by one line.

**Rule V-15.** The self-check runs first in every full invocation. A harness
that has not proven it can fail has not proven anything.
