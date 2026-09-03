# Cycle 0.14 — Scroll optimisation and performance

**The `CSI S` / `CSI L` / `CSI M` path that `specs/SCREEN_MODEL.md` R-25
deferred, plus the benchmark suite and its regression gate.**

## Why now and not earlier

R-25 defers it, and T-055 records why: scroll sequences are a large win for a
scrolling list and the single most common source of rendering bugs in mature
TUI libraries, because they require modelling the terminal's scrollback,
margins and clearing behaviour, all of which vary — and they turn a pure diff
into a stateful transformation of the front buffer.

**The gate is the whole point:** the optimisation lands only if the oracle can
prove that a scroll produces the same screen as a repaint. That instrument
exists from 0.7 and has been exercised for six cycles by the time this one
opens.

## Decisions in

T-055 (which this cycle supersedes with a new decision if the gate is met).

**Open questions to settle:** whether scroll optimisation ships at all. It is
legitimate to finish this cycle by recording that the gate was not met and the
optimisation stays out — that is a result, not a failure.

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| 0.14.0 | **The benchmark suite** — `harness/bench.py`, the five benchmarks, the committed baselines | numbers before any optimisation, which is the only way to know one helped |
| 0.14.1 | **The oracle's scroll model** — the mini-VT learns `CSI S`/`T`/`L`/`M` and the scrolling region | hand-written scroll sequences parsed into asserted screens |
| 0.14.2 | **Scroll detection** — recognising a shifted region in the diff | a detector that is conservative: it fires only where it is certain |
| 0.14.3 | **Emission and the front-buffer shift** — the sequences, and the matching transformation of `front` | the round trip green on scroll-shaped buffer pairs |
| 0.14.4 | **The decision** — ship it or record why not | a decision in `meta/DECISIONS.md`, either way |
| 0.14.5 | **Close** | `done/0.14/`, `0.15.0.md` written |

## Checklist

### 0.14.0 — benchmarks
- [ ] `harness/bench.py` writing one line per benchmark into `meta/bench/<date>.txt`
- [ ] the five: a full-screen repaint; a single-cell change; a 10 000-row list scrolled one row; the input decoder over a 1 MiB paste; the layout solver over a twelve-way split
- [ ] each reports both wall time and **emitted byte count** — the byte count is the number that matters for a terminal over a slow link and it is machine-independent
- [ ] the committed baseline, and the 20% regression gate against it on the same machine (V-12, V-13)
- [ ] the gate never compares absolute numbers across machines

### 0.14.1 — the oracle's scroll model
- [ ] `CSI Ps S` (scroll up), `CSI Ps T` (scroll down), `CSI Ps L` (insert line), `CSI Ps M` (delete line)
- [ ] `CSI t ; b r` (DECSTBM, the scrolling region) and its interaction with cursor positioning
- [ ] hand-written sequences parsed into asserted screens, before any renderer change
- [ ] the oracle still fails loudly on anything outside the extended subset

### 0.14.2 — scroll detection
- [ ] a detector over the (front, back) pair that finds a region shifted by `n` rows
- [ ] **conservative**: it fires only when the shifted region matches exactly and the residue is a single band. Anything else falls back to the ordinary diff, and the fallback is the safe direction
- [ ] a cost model deciding whether the scroll is cheaper than the repaint, in emitted bytes, with the threshold a named constant

### 0.14.3 — emission and the shift
- [ ] the sequences emitted with a scrolling region set and restored
- [ ] `front` transformed to match, so the next frame's diff is against the truth
- [ ] **the round trip on scroll-shaped pairs**: generate a buffer, shift it, render, parse, compare — ten thousand cases
- [ ] the ordinary random-pair corpus from 0.8.4 re-run and still green
- [ ] the benchmark showing the improvement, in bytes and in time

### 0.14.4 — the decision
- [ ] if the gate is met: a decision recording it, `specs/SCREEN_MODEL.md` R-25 superseded, T-055 annotated
- [ ] if it is not: a decision recording **why**, with the failing cases committed as goldens so a later attempt starts from evidence

## Gate

Ten thousand scroll-shaped round trips green **and** the ordinary corpus still
green **and** a measured improvement. Any one of the three missing means the
optimisation does not ship.

## Watch for

- **This is the cycle most likely to produce a wrong screen that looks
  plausible.** That is why it is gated on the oracle and why the detector is
  conservative.
- **The front-buffer shift is the subtle half.** A scroll that the terminal
  performed and the front buffer did not record makes every subsequent frame
  wrong, and the failure appears rows away from the cause.
