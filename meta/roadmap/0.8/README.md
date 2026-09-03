# Cycle 0.8 — The renderer

**`screen_render`: the diff, run splitting, cursor tracking, the pen,
synchronized output, and the zero-bytes rule.** The pure function at the centre
of the library.

## Decisions in

T-042, T-052, T-054, T-055. Settled.

**Open questions to settle:** O-R2 (`NTUI_GAP` = 8 and `NTUI_EL_MIN` = 4 —
confirmed by measurement here).

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| 0.8.0 | **The diff** — damaged rows, changed cells, run splitting at `NTUI_GAP` | a frame emitting only what changed |
| 0.8.1 | **Cursor tracking** — the exact model, the relative-form table, the wide-cell advance | the oracle's cursor and the renderer's agree on every golden |
| 0.8.2 | **The pen and the sequence table** — one place every escape sequence is spelled | `check_sequences_documented` live |
| 0.8.3 | **Frame open and close** — synchronized output, the reset, the cursor request, and the zero-bytes rule | an idle frame emits **nothing at all** |
| 0.8.4 | **The round trip** — random buffer pairs through render → oracle → compare | the strongest correctness statement the library makes |
| 0.8.5 | **Close** | `done/0.8/`, `0.9.0.md` written |

## Checklist

### 0.8.0 — the diff
- [ ] only damaged rows are visited; a clean row costs nothing
- [ ] within a row, changed cells found by comparison, then split into runs separated by gaps of at least `NTUI_GAP` unchanged cells
- [ ] the trailing-blank optimisation: `CSI K` when the tail is blank at the pen's background and at least `NTUI_EL_MIN` long
- [ ] **no `CELL_WIDE_TAIL` is ever emitted** (R-15)
- [ ] no cell emitted twice
- [ ] **no character emitted that did not come from a cell** (R-16): no padding spaces, no unrequested clearing, and **no `\n`, ever**
- [ ] O-R2 measured: `NTUI_GAP` and `NTUI_EL_MIN` confirmed against emitted-byte counts on a corpus of realistic frames, and the numbers recorded with the measurement

### 0.8.1 — cursor tracking
- [ ] the absolute form `CSI r ; c H`
- [ ] the relative forms from R-17: `CSI n C` for short right moves, `CR` for column 0, `CR` + `CSI B` for the next row — **never `\n`**
- [ ] the model is exact, including the two-column advance for a wide lead (R-18)
- [ ] a test that walks the emitted stream and asserts the implied cursor track matches the cells written
- [ ] autowrap is off, so nothing relies on it

### 0.8.2 — the pen and the sequence table
- [ ] every escape sequence the library can emit is spelled in `src/screen/seq.npk` and nowhere else
- [ ] `check_constants_named` extended to catch a literal outside it
- [ ] `check_sequences_documented` live: every sequence in that file appears in a specification table
- [ ] the pen applies T-042's transition rule from 0.5.2, unchanged

### 0.8.3 — frame open and close
- [ ] open: `CSI ? 2026 h` when capable, hide the cursor if shown, `CSI 0 m`, pen reset
- [ ] close: cursor to the application's request, show if asked, `CSI ? 2026 l`
- [ ] **synchronized output wraps every frame, unconditionally, in one place** (T-054)
- [ ] **a frame that resolves to no changes emits nothing at all** (R-21) — wrapper, reset and all. Asserted by a test that renders the same buffer twice and requires zero bytes the second time
- [ ] the last cursor request in a frame wins; no request means the cursor stays hidden (R-19)

### 0.8.4 — the round trip
- [ ] a generator producing random buffer pairs: random clusters from the 0.2 corpus, random styles, random damage patterns, random sizes including 1×1 and the maximum
- [ ] for each: render, parse with the oracle, compare the oracle's screen to the back buffer, cell for cell including styles and wide pairs
- [ ] **ten thousand pairs green** is the gate
- [ ] every failing pair minimised and committed as a permanent golden case
- [ ] the same corpus re-run after every later renderer change

### 0.8.5 — close
- [ ] `specs/SCREEN_MODEL.md`'s emitted-sequence tables reconciled against `seq.npk`
- [ ] the first widget-shaped golden cases written (a filled rect, a bordered box, a line of mixed-width text) so 0.11 starts with examples
- [ ] findings written; `0.9.0.md` written; archived

## Gate

**Ten thousand random buffer pairs surviving the round trip.** A renderer bug
that produces plausible bytes does not survive this, and nothing else in the
plan catches it.

## Watch for

- **This is where "right on a full repaint, wrong on a partial diff" lives** —
  the compiler's cycle-0.5 lesson in this library's terms. The random-pair test
  is the shape that finds it; a hand-written case set tests the paths the author
  thought of.
- **`end` is a keyword**, and a diff wants a run's `end` constantly. `hi` is
  the reserved spelling (`specs/BUILD.md` §7).
- **The renderer reads nothing** (T-052). If it needs a fact, that fact is a
  parameter. A single `mono_now()` here would end the determinism claim.
