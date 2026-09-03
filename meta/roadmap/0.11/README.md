# Cycle 0.11 — Core widgets

**`src/widget/`: `Block`, `Clear`, `Padding`, `Span`, `Line`, `Text`,
`Paragraph`, `List`, `Table`, `Tabs`.** The set every application needs.

## Decisions in

T-004, W-1 … W-15 in `specs/WIDGET_MODEL.md`. Settled.

## The rule that governs every subcycle

**Every widget ships with golden cases at three widths** — narrow enough to
clip, exact, and wide enough to leave slack — **and one case containing a wide
CJK character, an emoji with VS16, and a combining mark** (V-9). Those three are
the shear cases. A widget without them has not been tested, and the checklist
below does not repeat the requirement per widget: it applies to all of them.

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| 0.11.0 | **The trait and the builder convention** — `Widget`, chained setters, the bare constructor functions | two trivial widgets proving the shapes probe 06 and 07 verified |
| 0.11.1 | **`Block`, `Clear`, `Padding`** — borders, titles, the inner area | the six `LineSet`s, titles at three alignments, nested blocks |
| 0.11.2 | **Text values** — `Span`, `Line`, `Text`; alignment in display width | alignment correct with mixed-width content (W-11) |
| 0.11.3 | **`Paragraph`** — wrapping, scrolling, the control-character rules | `Wrap.None`, `Wrap.Word`, `Wrap.Char`, and `\t`/`\n`/`\r`/C0 handling |
| 0.11.4 | **`List`** — items, selection, highlight, offset, `ListState` | a 10 000-item list scrolled to every boundary |
| 0.11.5 | **`Table`** — columns with `Constraint`s, header, row selection, `TableState` | column widths from the 0.9 solver, not a second implementation |
| 0.11.6 | **`Tabs`** — a one-row selector | dividers, overflow, selection |
| 0.11.7 | **Close** | `done/0.11/`, `0.12.0.md` written |

## Checklist

### 0.11.0 — the trait and the convention
- [ ] `trait:Widget = { func:render = NIL(Self->:self, Buffer->:buf, Rect:area); };`
- [ ] `Self->` receiver, so an owning widget is not consumed per call (W-2)
- [ ] chained setters: `move Self` in, `Self` out (W-5)
- [ ] bare constructor functions, because there are no static methods (W-4)
- [ ] **no `Default` derive anywhere** (D-123 removed it, and a default that carries meaning is a value nobody chose)
- [ ] the sentinel test harness: render into a rect surrounded by a known pattern, assert it is untouched (W-3) — used by every widget from here on

### 0.11.1 — `Block`, `Clear`, `Padding`
- [ ] `LineSet` with twelve glyphs; `PLAIN`, `ROUNDED`, `DOUBLE`, `THICK`, `QUADRANT_INSIDE`, `QUADRANT_OUTSIDE`, `ASCII`
- [ ] border sides selectable individually
- [ ] titles: top and bottom, left/centre/right, multiple titles per edge
- [ ] `block_inner(area)` — the area inside the border, and it is what a caller draws into
- [ ] a block in a 1×1 area, a 2×2 area, and a 0-width area, none of which trap
- [ ] `Clear` blanks a region — the popup primitive
- [ ] **no automatic border joining** (W-13), and the reason recorded in the module header

### 0.11.2 — text values
- [ ] `Span` (styled string), `Line` (spans + alignment), `Text` (lines)
- [ ] alignment computed in **display width** (W-11), asserted with a line containing CJK, an emoji and a combining mark
- [ ] `Text` from a `string`, splitting on `\n`
- [ ] owning fields, so these are move-only — and the widget-builder chain accounts for it

### 0.11.3 — `Paragraph`
- [ ] `Wrap.None` clips; `Wrap.Word` breaks at the last space-class boundary that fits, falling back to the last cluster; `Wrap.Char` always breaks at the last cluster (W-9)
- [ ] "space class" is `U+0020`, `U+0009` and Unicode `Zs` — and the documentation says this is **not** UAX #14
- [ ] `\n` starts a line; `\t` advances to the next multiple of `tab_width`; `\r` is dropped; other C0/C1 render as control pictures or `?` (W-10)
- [ ] a scroll offset in lines and in columns
- [ ] a word longer than the area, wrapped at cluster boundaries, never mid-cluster
- [ ] a width-2 cluster with one column left becomes a blank (X-13), asserted

### 0.11.4 — `List`
- [ ] items as `Vec<Text>`; per-item styles; a highlight style and symbol
- [ ] `ListState { offset, selected }` owned by the application (W-7), fields public (W-8)
- [ ] `select_next` / `select_previous` / `select_first` / `select_last` as state helpers, and the scroll offset following the selection
- [ ] items taller than one row
- [ ] a 10 000-item list scrolled to top, bottom, and every boundary in between
- [ ] an empty list, and a list taller than its area, and an area of zero height

### 0.11.5 — `Table`
- [ ] columns with `Constraint`s solved by **`layout_split`** — not a second implementation (a tree check greps `src/widget/table.npk` for arithmetic that should be the solver's)
- [ ] a header row, optionally styled and optionally sticky
- [ ] `TableState { offset, selected }`
- [ ] per-cell styles, column spacing, and a highlight that spans the row
- [ ] a table whose columns do not fit, resolved by the solver's deficit rule

### 0.11.6 — `Tabs`
- [ ] titles, a divider, a selection style
- [ ] overflow behaviour when the titles do not fit — stated, tested, and the same every time

## Gate

Every widget has its three-width and its mixed-script golden cases, the
sentinel test passes for all of them, and `Table` provably uses the layout
solver rather than its own arithmetic.

## Watch for

- **A widget that compiles is not a widget that renders.** This is the
  compiler's cycle-0.4 lesson in this library's terms, and the golden oracle is
  the sweep that answers it. Write the golden case *with* the widget, not after
  the set is finished.
- **`Text` and `Line` own strings**, so they are move-only. The builder chain
  (`move Self` in, `Self` out) is what makes that bearable; a setter taking
  `Self->` would fight it.
- **Alignment in bytes is the most likely bug in this cycle** and the mixed-script
  case is the only test that finds it.
