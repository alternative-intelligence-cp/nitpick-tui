# Layout

Rectangles, and a deterministic integer constraint solver. This is the part of
a TUI that is arithmetic all the way down, which in this language means it is
the part the compiler is best at and the part where an overflow traps rather
than producing a box in the wrong place.

---

## 1. `Rect`

```nitpick
pub struct:Rect = { uint16:x; uint16:y; uint16:w; uint16:h; };
```

**Rule L-1.** `uint16`, so a coordinate cannot be negative and cannot be
spelled negative (`SAFETY.md` S-14). `x + w` and `y + h` are computed in
`uint32` and clamped, because `x + w` genuinely can exceed 65535 for a rect
that has been offset, and D-210 makes the unclamped version a trap.

**Rule L-2 — `w == 0` or `h == 0` is an empty rect and is legal everywhere.**
Every operation accepts one, every widget renders nothing into one, and no
function fails because a rect is empty. Emptiness is the natural result of a
window too small for a layout, which is an everyday condition and not an error.

**Rule L-3 — the operations are total.**

| Operation | Result |
|---|---|
| `rect_intersect(a, b)` | the overlap, or an empty rect at `(a.x, a.y)` when they do not overlap |
| `rect_union(a, b)` | the bounding box; the other rect when one is empty |
| `rect_contains(a, x, y)` | `false` for an empty rect |
| `rect_inner(a, margin)` | shrunk by the margin, clamped at empty — never inverted |
| `rect_clamp(a, bound)` | `a` moved and shrunk to fit inside `bound` |
| `rect_offset(a, dx, dy)` | translated, saturating at 0 and 65535 |
| `rect_area(a)` | `uint32`, so a full screen's area does not overflow |

"Clamped at empty, never inverted" is the load-bearing half: a margin larger
than the rect gives `w = 0`, not `w = 65530` from an underflow. In this
language that underflow would trap rather than corrupt, which is better, but a
trap in a layout calculation is still a program that stopped because a window
got small.

---

## 2. Constraints

```nitpick
pub enum:Constraint = {
    Fixed(uint16);          // exactly n cells
    Percent(uint16);        // n% of the available extent, 0..100
    Ratio(uint16, uint16);  // num/denom of the available extent
    Fill(uint16);           // a share of the leftover, by weight
    Min(uint16);            // at least n; behaves as Fill(1) above that
    Max(uint16);            // at most n; takes what it can up to n
};
```

**Rule L-4 — no floating point anywhere.** `Percent` and `Ratio` are integer
divisions of the available extent, and the remainder is distributed by §4's
rule. Cassowary-style simplex over floats — what several widely used libraries
do — gives results that depend on iteration order and can differ by a cell
between builds. A layout that differs by a cell between builds is a golden test
that cannot exist.

---

## 3. The split

```nitpick
pub struct:Layout = {
    Direction:dir;             // Horizontal | Vertical
    Margin:margin;             // outer
    uint16:gap;                // between adjacent segments
    Justify:justify;           // where slack goes when nothing fills, §5
    Vec<Constraint>:parts;
};

pub func:layout_split = Vec<Rect>(Layout->:l, Rect:area);
```

`layout_split` returns one `Rect` per constraint, in order, always — including
empty ones. A caller indexes the result positionally and never has to ask which
constraint a rect came from.

---

## 4. The algorithm

Stated completely, because "deterministic" is only a claim if the algorithm is
written down.

Let `T` be the extent along `dir` of `area` after the margin, `n` the number of
constraints, `g` the gap.

**Step 1 — the gap budget.** `avail = T - g * (n - 1)`, computed in `uint32`. If
`n == 0`, return empty. If `g * (n - 1) >= T`, every segment is empty and the
result is `n` empty rects at the area's origin.

**Step 2 — the desired size of each non-`Fill` segment**, in declaration order:

| Constraint | Desired | Floor | Ceiling |
|---|---|---|---|
| `Fixed(k)` | `k` | `k` | `k` |
| `Percent(p)` | `avail * min(p, 100) / 100` | 0 | `avail` |
| `Ratio(a, b)` | `avail * a / b` (b = 0 → 0) | 0 | `avail` |
| `Max(k)` | `k` | 0 | `k` |
| `Min(k)` | `k` | `k` | `avail` |
| `Fill(w)` | — | 0 | `avail` |

All products are computed in `uint32` and the divisions truncate.

**Step 3 — the leftover.** `used = sum(desired)` over non-`Fill` segments;
`left = avail - used` when `used <= avail`, else a deficit of `used - avail`.

**Step 4a — distributing a surplus.** Let `W = sum(w)` over `Fill(w)` segments
plus `sum(1)` over `Min` segments that received their floor (a `Min` grows
above its floor like `Fill(1)` — that is what distinguishes it from `Fixed`).

- If `W > 0`: each gets `floor(left * w_i / W)`, and the remainder
  `left - sum(...)` is distributed **one cell each to the first segments in
  declaration order**. Declaration order, not largest-remainder: largest-
  remainder is fairer and is a tie-break away from being non-deterministic, and
  a layout that gives the extra cell to the leftmost pane every time is a
  layout a user can predict.
- If `W == 0`: the surplus is **slack**, positioned by `justify` (§5).

**Step 4b — resolving a deficit.** Shrink in **reverse declaration order**,
each segment down to its floor, until the deficit is gone. If a deficit
remains after every segment is at its floor, shrink again in reverse order to
zero. The first-declared segment is the last to lose space, which matches how
people write layouts — the important thing first.

**Step 5 — positions.** `pos_0 = area.origin + margin`; `pos_{i+1} = pos_i +
size_i + g`. The cross-axis extent of every segment is the area's, minus the
margin.

---

## 5. `Justify`

```nitpick
pub enum:Justify = { Start; End; Center; SpaceBetween; SpaceAround; };
```

Applies only when step 4a found no filler. `SpaceBetween` distributes the slack
into the `n - 1` gaps by the same floor-then-first-in-order rule as step 4a;
`SpaceAround` uses `n + 1` gaps. `Start` is the default and puts all slack at
the end.

---

## 6. The invariants

**Rule L-5.** For every input, `layout_split` guarantees:

1. it returns exactly `n` rects;
2. every returned rect is inside `area` — `x >= area.x`, `x + w <= area.x +
   area.w`, and the same vertically;
3. no two returned rects overlap;
4. they are ordered along `dir`: `pos_i + size_i <= pos_{i+1}`;
5. the cross-axis extent of every rect equals the area's, less the margin;
6. `sum(size_i) + g * (n - 1) <= T` — the layout never overflows its area.

**Rule L-6.** These six are the layout engine's contribution to cycle 1.5's
obligation set (`VERIFICATION.md` §4). They are written now, as `ensures`
clauses on `layout_split`, refused by the compiler's rung until 1.5.3 lands
contracts, and discharged then. Until then they are a property test over a
generated corpus of constraint sets — which is weaker, and is what makes
writing them down now worth doing.

**Rule L-7.** `Fixed(k)` with `k > avail` is clamped, not an error. A pane that
asks for eighty columns in a forty-column terminal gets forty. `ETuiGeometry`
is reserved for a request that is *incoherent* rather than merely
unsatisfiable — a rect outside its parent, a `Ratio` with a zero denominator
where the caller asked for strict checking — and the ordinary too-small case is
an answer.

---

## 7. Composition

**Rule L-8.** Layouts nest by splitting a returned rect again. There is no
layout tree, no parent pointers, and no invalidation: a split is a function
call and a frame calls it as many times as it needs.

That is the immediate-mode consequence and it is a real cost — a deep layout is
recomputed every frame. It is affordable: a split is a few dozen integer
operations and a screen has tens of them. The alternative is a retained tree
with a dirty protocol, which is a source of the "why did this not update" class
of bug, and which this library's architecture decision (T-004) declined for
exactly that reason.

**Rule L-9 — `layout_split` allocates**, returning a `Vec<Rect>`. A caller in a
hot path uses `layout_split_into(l, area, Vec<Rect>->:out)`, which clears and
refills a caller-owned vector, so a frame can reuse one. Both exist; the
allocating one is the one examples use.

---

## 8. Open items

- **O-L1 — whether `Min` should grow like `Fill(1)` or not grow at all.** As
  specified it grows, which matches every flexbox-shaped system and is what
  makes `[Min(10), Fixed(3)]` do the obvious thing. The alternative reading —
  `Min` takes exactly its floor unless something else forces it larger —
  produces layouts with unexpected trailing gaps. Recorded because the two
  readings both occur in existing libraries and the choice must be one.
