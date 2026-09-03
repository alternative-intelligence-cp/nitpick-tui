# Cycle 0.9 — Layout

**`src/layout/`: `Rect`, `Constraint`, `Layout`, `layout_split`, and the six
invariants.** Integer arithmetic with a written algorithm and a written
remainder rule.

## Decisions in

T-060, T-061. Settled.

**Open questions to settle:** O-L1 (does `Min(k)` grow above its floor?
recommendation: yes).

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| 0.9.0 | **`Rect`** — the seven total operations | every operation total, empty rects legal everywhere |
| 0.9.1 | **The solver** — the five steps of `specs/LAYOUT_MODEL.md` §4 | the algorithm as written, not an approximation of it |
| 0.9.2 | **The invariants** — the six, as property tests and as written `ensures` | a generated corpus of constraint sets, all six green |
| 0.9.3 | **`Justify` and nesting** — slack placement, `layout_split_into` | the flexbox-shaped cases |
| 0.9.4 | **Close** | `done/0.9/`, `0.10.0.md` written |

## Checklist

### 0.9.0 — `Rect`
- [ ] `uint16` fields; `x + w` computed in `uint32` and clamped (L-1)
- [ ] `rect_intersect`, `rect_union`, `rect_contains`, `rect_inner`, `rect_clamp`, `rect_offset`, `rect_area`
- [ ] **every one total** (L-3): a non-overlapping intersection is an empty rect at `(a.x, a.y)`, a margin larger than the rect gives `w = 0` and never an inverted rect
- [ ] an empty rect is legal in every operation and no function fails because of one (L-2)
- [ ] `rect_area` returns `uint32`
- [ ] a property test over random rects asserting totality and the clamping rules

### 0.9.1 — the solver
- [ ] the five steps exactly as `specs/LAYOUT_MODEL.md` §4 writes them
- [ ] the gap budget, including `g * (n-1) >= T` giving `n` empty rects
- [ ] the desired-size table, all six constraints, products in `uint32`, divisions truncating
- [ ] surplus: proportional shares, remainder **one cell each to the first segments in declaration order** — never largest-remainder (T-061)
- [ ] deficit: shrink in **reverse** declaration order to each floor, then to zero
- [ ] positions: `pos_{i+1} = pos_i + size_i + g`
- [ ] `Ratio` with a zero denominator is 0, not a division
- [ ] `Percent` above 100 clamps
- [ ] O-L1 decided and recorded
- [ ] **no floating point anywhere** — a tree check greps for `flt32`/`flt64` in `src/layout/`

### 0.9.2 — the invariants
- [ ] the six from L-5 as property tests over a generated corpus: every constraint kind, counts 0…16, extents 0…512, gaps 0…8, all four `Justify` values
- [ ] the same six written as `ensures` clauses in comment form, in the syntax they will take (`specs/VERIFICATION.md` §4, P-1)
- [ ] the `requires (p <= 100)` and `requires (b != 0)` preconditions likewise
- [ ] a test that constructs an over-full layout and asserts the deficit resolution order
- [ ] a test that constructs an under-full layout with no filler and asserts the slack goes where `Justify` says

### 0.9.3 — `Justify` and nesting
- [ ] `Start`, `End`, `Center`, `SpaceBetween`, `SpaceAround`
- [ ] `SpaceBetween` over `n-1` gaps and `SpaceAround` over `n+1`, both by the same floor-then-first-in-order rule
- [ ] `layout_split_into` refilling a caller-owned `Vec<Rect>` (L-9)
- [ ] nested splits: a returned rect split again, three deep, with the invariants holding at every level

## Gate

The six invariants green over the generated corpus, and no floating point in
the module.

## Watch for

- **`limit` is the verification keyword** and this is the module that wants it
  most. `bound` is the reserved spelling.
- **Determinism is the whole reason for T-061.** A Cassowary solver over floats
  would be shorter to write and would make every golden test in cycles 0.11–0.13
  impossible.
- **The remainder rule is user-visible.** Giving the extra cell to the leftmost
  pane every time is a layout a person can predict; largest-remainder is fairer
  and unpredictable, and unpredictable loses.
