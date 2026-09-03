# Cycle 0.7 — The oracle

**`tests/vt/`: a miniature VT parser in Nitpick, and the golden harness stage.**
The instrument that makes every later cycle's rendering claims checkable.

## Why before the renderer

Because an instrument co-developed with the thing it judges tends to agree with
it. Written first and tested against **hand-written byte sequences**, the
oracle is a working checker on the day the renderer's first line is written —
so the renderer is developed against it rather than beside it.

This is the compiler's own ordering rule ("instruments precede the constructs
they guard") applied to the highest-value instrument this library has.

## Decisions in

T-091, T-092, and V-5 … V-9 in `specs/TESTING.md`. Settled.

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| 0.7.0 | **The VT parser** — the sequence subset, the cell grid, the loud failure on anything unrecognised | hand-written sequences parsed into asserted screens |
| 0.7.1 | **The screen dump format** — a committed text rendering with a style legend, diffable by a human | a screen file a reviewer reads in a pull request |
| 0.7.2 | **The `golden` stage** — three artifacts per case, `--record-golden`, the refusal to record during an ordinary run | the two pending self-check cases from 0.0.3 go live |
| 0.7.3 | **Close** | `done/0.7/`, `0.8.0.md` written |

## Checklist

### 0.7.0 — the VT parser
- [ ] parses exactly the subset `specs/SCREEN_MODEL.md` says the renderer may emit: `CUP`, `CUU`/`CUD`/`CUF`/`CUB`, `CR`, `ED`, `EL`, `SGR` (all forms including colon sub-parameters), the DEC private modes the library sets, OSC 8, and UTF-8 text
- [ ] **fails loudly on any sequence it does not recognise** (V-6) — this is what keeps the emitted-sequence table honest
- [ ] a cell grid with the same wide-lead/tail model as `Buffer`, so a comparison is field for field
- [ ] autowrap **off**, matching the renderer's world; a write past the last column is an error the oracle reports, not a silent wrap
- [ ] tracks the cursor exactly, including the two-column advance for a wide cluster
- [ ] tracks the pen, so a missing style reset shows up as a styled cell
- [ ] **tested against hand-written sequences**: at least forty cases, each a byte string and an expected screen, covering every sequence in the subset
- [ ] a negative suite: sequences the oracle must reject, including one the renderer could plausibly emit by mistake

### 0.7.1 — the screen dump format
- [ ] a text rendering: one line per row, one character per cell (the cluster, or a placeholder for a wide tail), with a style legend below keyed by letter
- [ ] deterministic: the same screen always dumps identically, legend order included
- [ ] a human can read a diff of two dumps and see what changed — the format is judged by that and by nothing else

### 0.7.2 — the `golden` stage
- [ ] three artifacts per case: the program, the expected bytes, the expected screen (V-7)
- [ ] the stage: build, run, capture bytes, compare; parse with the oracle, dump, compare
- [ ] `--record-golden` as a deliberate act, refused during an ordinary run (V-8)
- [ ] self-check cases 4 and 5 from 0.0.3 go live: a one-byte difference fails, and — the important one — **bytes that match with a screen that differs fails**
- [ ] a golden case whose bytes the oracle cannot parse fails with the oracle's message, naming the sequence

## Gate

The self-check's case 5 — matching bytes, differing screen — fails, proving the
round trip is doing work that byte equality does not.

## Watch for

- **The oracle is not the library.** It lives in `tests/vt/`, imports nothing
  from `src/`, and shares no code — a shared bug would agree with itself. It
  may import `src/core/` for `Vec` and `Bytes`; that is storage, not semantics,
  and the boundary is written down.
- **A loud failure on an unrecognised sequence will be annoying**, repeatedly,
  in 0.8 and 0.14. That is the feature.
