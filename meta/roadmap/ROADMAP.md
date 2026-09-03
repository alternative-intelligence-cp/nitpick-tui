# Roadmap — the cycle map

The specification set (`meta/specs/`) is written and the decisions it rests on
are in `meta/DECISIONS.md`. This is the plan built on them.

## How this is organised

- **A cycle is a folder** — `0.0/`, `0.1/`, … — focused on **one topic**.
- **A subcycle is a file inside it** — `0.0.0.md`, `0.0.1.md`, … — one workable
  chunk of that topic, written execution-grade before its code is touched.
- **A finished cycle moves to `done/`**, so the active work stays easy to find.
- **Commit after every subcycle. Push at the end of every cycle.**
- **Every cycle's README carries a checklist.** Tick items as they land; the
  checklist is the cycle's state, and a cycle whose checklist is complete is a
  cycle ready to close.

This convention is the compiler repository's, deliberately, so that a session
moving between the two reads one thing.

**Every cycle is planned in more detail than the compiler's roadmap plans its
later ones**, and that is a departure with a reason: this library plans against
a *frozen* language and against published terminal protocols, not against a
design that later cycles will teach us to change. Where a cycle genuinely
depends on what an earlier one measures, the plan says so and leaves the
measurement, not a guess. The parts that are still guesses are in
`../OPEN_QUESTIONS.md` with recommendations, not hidden in a subcycle file.

---

## The constraint that shapes everything

**`ntui` cannot be built by `npkg` today**, and cross-repository imports do not
resolve (`specs/BUILD.md` §1, `../OPEN_QUESTIONS.md` O-N2). Consequently:

> **`harness/` is the build and test runner, and every import is relative,
> until the compiler's tooling closes O-N2.**

That is the same relationship `bootstrap/harness/` has to `npkg` in the
compiler repository, and it retires the same way. It is the first thing cycle
0.0 builds, because everything after it is tested by it.

The second constraint, and the one that shapes the *architecture*:

> **The language has no closures (D-018), `defer` does not run on a trap
> (D-014), and every public `error:` costs every consumer a `failsafe` arm
> (REACH-002).**

The first chose the application model (T-004). The second made terminal
restoration a `failsafe`-reachable, allocation-free operation (T-012), which is
this library's headline safety property. The third capped the error budget at
nine (T-018) and decided the module decomposition.

---

## Phase 0 — the library, built bottom-up

Nothing here is user-facing until 0.10. Each cycle is testable on its own,
which is the point of the ordering: the headless device (T-090) means every
cycle from 0.2 onward is exercised with no terminal at all.

| Cycle | Topic | Gated on |
|---|---|---|
| **0.0** | **Foundations** — the language probes, the harness, `src/core/` | — |
| **0.1** | **The device** — `/dev/tty`, termios, size, signals, the restore record, suspend/resume, the headless device | 0.0 |
| **0.2** | **Text** — UTF-8, the generated Unicode tables, width, grapheme segmentation | 0.0 |
| **0.3** | **Input** — the decoder, every protocol, the fixture corpus, the fuzzer | 0.0, 0.2 |
| **0.4** | **Capabilities** — the table, the runtime probe, the width calibration | 0.1, 0.3 |
| **0.5** | **Style** — colour, attributes, the degradation tables, SGR emission | 0.0 |
| **0.6** | **The screen model** — `Cell`, `Buffer`, damage, the drawing primitives | 0.2, 0.5 |
| **0.7** | **The oracle** — the mini-VT and the golden harness, tested against hand-written sequences | 0.6 |
| **0.8** | **The renderer** — the diff, cursor tracking, synchronized output | 0.6, 0.7 |
| **0.9** | **Layout** — `Rect`, constraints, the solver, the six invariants | 0.0 |
| **0.10** | **The runtime** — the loop, timers, the post channel, `Program` and `Ctx` | 0.1, 0.3, 0.8 |
| **0.11** | **Core widgets** — `Block`, `Line`, `Paragraph`, `List`, `Table`, `Tabs`, `Clear` | 0.7, 0.9 |
| **0.12** | **Data and graphics widgets** — `Gauge`, `Sparkline`, `Scrollbar`, `Canvas`, `Chart`, `BarChart`, `Tree` | 0.11 |
| **0.13** | **Input widgets** — `TextInput`, `TextArea` | 0.11, 0.3 |
| **0.14** | **Scroll optimisation and performance** — the deferred `CSI S`/`L`/`M` path, benchmarks, the regression gate | 0.8, 0.12 |
| **0.15** | **The dogfood application** — a real program, written against the library by someone using it rather than writing it | 0.13 |
| **0.16** | **Hardening** — the fuzz sweep, the stress sweep, the compatibility matrix, the verification obligations | 0.15 |
| **1.0** | **Release** — documentation, the API freeze, versioning, the `failsafe` arm contract | 0.16 |

---

## What each cycle produces

Enough detail that a reader can tell whether a cycle is finished without
opening it. The per-cycle README has the subcycle map and the checklist.

### 0.0 — Foundations
The **language probes** first: fourteen small programs that verify the compiler
accepts the shapes every later cycle depends on — a struct holding a borrow
passed down and built by literal (T-056), a POD array of 32-byte values, a
tagged enum with payloads in a `pick`, `move` semantics on a generic container,
an inherent impl on a generic struct, a `Self->` trait receiver, an `async`
trait method behind a generic bound, an eventfd through `sys`, a struct laid
out in a `buffer` and handed to `ioctl`. **A probe that fails changes the
design, and finding that out now costs a day where finding it out in 0.11
costs a cycle.**

Then the harness (`harness/run.py`, its self-check, the stages), the manifest's
test table, CI, and `src/core/` — `Vec<T>`, `Bytes`, `SmallMap<K,V>`, and the
one file of named limits.

### 0.1 — The device
`src/term/`: the syscall vocabulary, `/dev/tty` with the inherited fallback,
the exact raw-mode edit, the restore record and `ntui_restore()`,
`TIOCGWINSZ`, `signalfd`, suspend/resume, and the headless device. The
`device` test stage, which is the only stage that needs a real terminal.

**The cycle's own gate:** a program that enters raw mode, traps deliberately,
and leaves a usable terminal — verified by a script that captures `stty -a`
before and after and requires them equal.

### 0.2 — Text
`tools/gen_unicode.py`, the committed tables, the regeneration check, the UTF-8
decoder and encoder, `cluster_width`/`text_width`, and the UAX #29 segmenter
with its carried state. The Unicode version is pinned here (Q-1).

**The cycle's own gate:** the segmenter run over the UCD's own
`GraphemeBreakTest.txt`, every case, no exceptions.

### 0.3 — Input
`src/input/`: the state machine, every sequence in `specs/INPUT_MODEL.md`, the
table-driven suite run twice (whole and byte-at-a-time), the fixture capture
tool, the first fixture directories, and the fuzzer.

**The cycle's own gate:** every fixture decodes to the event its filename
names, and the fuzzer runs a million inputs without a trap.

### 0.4 — Capabilities
`src/caps/`: `tools/caps.toml`, the generated table, the environment ladder,
the probe with DA1 as its sentinel, the width calibration, `NTUI_CAPS`, and
`ntui_caps_explain()`.

**The cycle's own gate:** the probe run against three real terminals, with the
transcripts committed, and a fourth terminal that answers nothing resolving to
the table row without hanging.

### 0.5 — Style
`src/style/`: `Color`, `Style`, `StylePatch`, the attribute table, the
degradation tables and their generator, and the SGR emitter with T-042's
transition rule.

**The cycle's own gate:** the RGB→256 mapping walked over all 2^24 inputs
against its invariants, and the patch algebra's associativity over a generated
corpus.

### 0.6 — The screen model
`src/screen/`: `Cell` with its measured inline size, `Buffer` with the cluster
pool and the link table, per-row damage, the clipping `Surface` primitives, and
the wide-character pairing rules.

**The cycle's own gate:** a property test that writes random clusters at random
positions and asserts the wide-pair invariants hold after every write.

### 0.7 — The oracle
`tests/vt/`: a miniature VT in Nitpick that parses the sequences
`specs/SCREEN_MODEL.md` says the renderer may emit, into a cell grid, failing
loudly on anything else. Plus the golden harness stage, `--record-golden`, and
the self-check cases that prove a golden mismatch and a *screen* mismatch both
fail.

**Written before the renderer, deliberately**, and tested against hand-written
byte sequences so that the renderer is developed against an oracle that already
works.

### 0.8 — The renderer
`screen_render`: the diff, the run splitting at `NTUI_GAP`, the trailing-blank
optimisation, exact cursor tracking, the pen, synchronized output, and the
zero-bytes-when-nothing-changed rule.

**The cycle's own gate:** the round trip (T-091) green on a corpus of
randomly generated buffer pairs — render, parse with the oracle, compare to the
back buffer — which is the strongest statement this library can make about its
own correctness.

### 0.9 — Layout
`src/layout/`: `Rect` and its total operations, `Constraint`, `Layout`,
`layout_split` and `layout_split_into`, `Justify`, and the six invariants as
property tests with the `ensures` clauses written as comments in the syntax
they will take (`specs/VERIFICATION.md` P-1).

### 0.10 — The runtime
`src/app/`: the `Program` trait, `Ctx`, the loop with its single multi-fd wait,
the timer wheel, the eventfd post channel, the redraw policy, `run`, and the
replay recorder. The first end-to-end program.

### 0.11–0.13 — Widgets
In three cycles because the sets have different dependencies and different test
shapes. Every widget ships with golden cases at three widths and one containing
a wide CJK character, an emoji with VS16, and a combining mark (T-091's V-9).

### 0.14 — Scroll optimisation and performance
The `CSI S`/`CSI L`/`CSI M` path that `specs/SCREEN_MODEL.md` R-25 deferred,
**gated on the oracle proving a scroll produces the same screen as a repaint**.
Plus `harness/bench.py`, the committed baselines, and the 20% regression gate.

### 0.15 — The dogfood application
A real program in `examples/`, substantial enough to find what examples do not:
a log viewer with search and follow, or a process monitor. Written against the
library as a consumer, with every friction recorded as a finding.

### 0.16 — Hardening
The fuzz corpus swept to exhaustion, `// stress: 40` on everything with a
timing dimension, the compatibility matrix filled with captured fixtures, and
`specs/VERIFICATION.md`'s obligation list reconciled against what the code
actually generates.

### 1.0 — Release
`docs/` written, the public API frozen and enumerated, the `failsafe` arm
contract published per module, examples for every widget, and the version
policy from Q-4.

---

## Post-1.0, as a map rather than a plan

| Cycle | Topic |
|---|---|
| **1.1** | Image protocols — Kitty graphics, Sixel, iTerm2 inline (Q-3) |
| **1.2** | The retained layer — an optional component tree above the immediate core, if 0.15 and 1.0's consumers ask for it |
| **1.3** | `aarch64` Linux — the syscall numbers differ, the structures do not (T-008) |
| **1.4** | Verified build — `ntui`'s obligations through the compiler's `npkg verify`, once that reaches libraries |

---

## Ordering notes

- **The probes come first, in 0.0, not last.** Fourteen small programs that ask
  the compiler whether the design is spellable. The compiler's own experience
  is the argument: *a construct that parses is not a construct that works*, and
  three of its cycles were mostly repair to constructs an earlier cycle had
  "finished". T-056 exists because one of these probes is expected to be
  interesting.
- **The harness comes first too**, for the compiler's stated reason about
  diagnostics: it is how every later cycle is tested, and a suite written after
  the code is a suite shaped by the code.
- **The device is early because it is the riskiest**, not because much depends
  on it. Everything above it uses the headless device. If `/dev/tty`, the
  restore record on the trap path, or `signalfd` turn out not to work as
  specified, that is a cycle-0.1 problem and not a cycle-0.10 surprise.
- **The oracle precedes the renderer.** Writing the checker first, and testing
  it against hand-written sequences, means the renderer is developed against an
  instrument that already works rather than co-developed with its own judge.
- **Capabilities need the input decoder**, because a terminal's replies are
  events (T-034). That is why 0.1 builds the device *without* the probe and 0.4
  adds it.
- **Scroll optimisation is gated on the oracle**, not scheduled. It is the
  single most common source of rendering bugs in mature TUI libraries and the
  only defensible way to add it is with something that can prove the result.
- **Verification obligations are written from cycle 0.0 onward**, in the syntax
  they will take, enforced by property tests until the compiler's 1.5 makes
  them real. The compiler's R9 is explicit: obligations discovered in a branch
  and never collected are the cheapest way to lose the campaign.
- **A decision precedes the cycle that needs it.** Each cycle's README lists
  its open questions; a cycle whose questions are open is not ready to start,
  and the plan says so rather than discovering it at the first subcycle.

---

## What to expect, from the compiler's experience

Four findings the compiler project paid for. They are recorded here because
they are about how work of this shape goes wrong, and none of them is specific
to a compiler.

**A construct that parses is not a construct that works.** Most of the
compiler's cycle 0.4 was repair, and every repair dated to the cycle that had
parsed the construct. Here: a widget that compiles is not a widget that
renders, and the golden oracle is the analogue of the sweep that found those.

**An analysis that is right on straight-line code and wrong after a merge
passes every test written the easy way.** Here: a renderer that is right on a
full repaint and wrong on a partial diff, and a decoder that is right on a
whole sequence and wrong when a read splits it. Both have a dedicated test
shape in the plan — random buffer pairs, and every fixture fed one byte at a
time — because neither is found by testing the easy way.

**Every hole was found by a check that diffs two lists, and none by a test.**
`specs/TESTING.md` §3 is this library's list of such checks, and the plan
schedules each one in the cycle that creates what it diffs.

**A suite that only ever agrees with what it is handed is worse than no
suite.** The harness self-check (V-14) is not optional and runs first.

---

## The cycle-numbering convention

Cycle numbers sort lexically only up to `0.9`; `0.10` sorts before `0.2` in a
plain listing. The compiler hit this and chose correctness over comfort, and so
does this plan: **the table above is authoritative over lexical order.** The
alternative — renumbering to keep single digits — would invalidate every
cross-reference the moment a cycle is inserted, which is a cost this project
has watched the compiler pay twice.
