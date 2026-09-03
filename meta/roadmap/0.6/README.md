# Cycle 0.6 — The screen model

**`src/screen/`: `Cell`, `Buffer`, the cluster pool, the link table, per-row
damage, and the clipping drawing primitives.** The data model the renderer
diffs and the widgets write into.

## Decisions in

T-050, T-051, T-053, T-056. Settled.

**Open questions to settle:** O-R1 (the inline cluster size — **a measurement**,
made in 0.6.0).

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| 0.6.0 | **`Cell`** — the layout, the measurement that sets the inline size, the flags, the blank rule | `#size_of<Cell>() == 32` asserted, and the inline size recorded with its corpus |
| 0.6.1 | **`Buffer`** — the grid, the accessor pair, the cluster pool, the link table, resize | one accessor pair, and nothing outside the module indexing storage |
| 0.6.2 | **Wide characters** — lead and tail, the overwrite rules, the last-column rule | the pairing invariants proven by a property test over random writes |
| 0.6.3 | **Damage** — the per-row span, the widen-on-write rule | damage over-approximates and the diff resolves; asserted both ways |
| 0.6.4 | **Drawing primitives** — `Surface` and the seven operations in `specs/SCREEN_MODEL.md` §9 | a widget-shaped write that computes past its edge is clipped, not an error |
| 0.6.5 | **Close** | `done/0.6/`, `0.7.0.md` written |

## Checklist

### 0.6.0 — `Cell`
- [ ] **the measurement**: a corpus of real terminal content (source code, prose in several scripts, emoji, box drawing) segmented into clusters, and the byte-length distribution recorded. O-R1's answer is the number that covers it, and the number is written into `meta/DECISIONS.md` with the corpus
- [ ] `Cell` is exactly 32 bytes; `#size_of` asserted
- [ ] **no owning field** — `check_no_owning_cells` covers it
- [ ] `len == 255` means spilled, and the first four bytes are a pool index
- [ ] `len == 0` is blank, renders as a space, and compares equal to another blank of the same style **regardless of `text`** (R-3) — one comparison function, and it is the authority
- [ ] the three flags: `CELL_WIDE_LEAD`, `CELL_WIDE_TAIL`, `CELL_SKIP`

### 0.6.1 — `Buffer`
- [ ] `buffer_at` and `buffer_set` are the only things that index `cells.items` — enforced by a tree check
- [ ] their bounds contracts written in the syntax they will take (`specs/VERIFICATION.md` P-2), with property tests standing in
- [ ] the cluster pool: `Bytes`, cleared with the buffer, offsets valid within the frame
- [ ] the link table: index 0 reserved, `ETuiCapacity` on overflow
- [ ] resize reallocates and **clears** (R-8), marks everything damaged, and does not attempt to preserve content
- [ ] `buffer_clear` resets cells, pool and links together

### 0.6.2 — wide characters
- [ ] a width-2 cluster writes a lead and overwrites the next cell as a tail of the same style (R-4)
- [ ] writing over a tail blanks its lead; writing over a lead blanks its tail (R-5)
- [ ] a width-2 cluster in the last column becomes a blank (R-6), and the fact is recorded for a debug report
- [ ] **the property test**: random clusters at random positions, asserting after every write that no tail is orphaned, no lead is untailed, and no cell is both
- [ ] the same test with clusters drawn from the 0.2 corpus, so real emoji and CJK are in it

### 0.6.3 — damage
- [ ] `RowDamage { lo, hi }` with `hi < lo` meaning clean
- [ ] a write widens the span even when the value is unchanged (R-13)
- [ ] a test asserting damage is an over-approximation: a write of the same value marks damage and the diff emits nothing
- [ ] a test asserting nothing outside a damaged span ever differs

### 0.6.4 — drawing primitives
- [ ] `Surface` built **by struct literal** at every site, never returned from a function (T-056) — a tree check greps for a function whose return type is `Surface`
- [ ] the seven operations in §9: `set_cluster`, `set_str`, `fill`, `clear`, `set_style`, `hline`/`vline`, and the sub-surface literal idiom
- [ ] a write outside the surface is **silently clipped**, not an error (R-22)
- [ ] the sentinel test: draw into a rect surrounded by a known pattern, assert the pattern is untouched
- [ ] `set_str` returns columns consumed, measured by 0.2's width function
- [ ] `LineSet` and the six shipped sets (`specs/WIDGET_MODEL.md` W-12) live here, not in the widget layer, because `hline`/`vline` take one

## Gate

The wide-pair property test over random writes, and the sentinel test proving
clipping.

## Watch for

- **`buffer` is a type keyword** and this is the module that wants it as a
  name. `Buffer` (the type) and `body`/`cells` (the fields) are the spellings.
- **The blank rule (R-3) is subtler than it looks.** Two cells that render
  identically must compare equal or the diff emits work that changes nothing —
  and `text` bytes left over from a previous frame are exactly the case.
- **`Cell` having no owning field is not negotiable.** If the measurement in
  0.6.0 suggests clusters need more than fits, the answer is a bigger inline
  array or a better pool, never a `string`.
