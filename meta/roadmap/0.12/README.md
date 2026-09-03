# Cycle 0.12 — Data and graphics widgets

**`Gauge`, `LineGauge`, `Sparkline`, `Scrollbar`, `Canvas`, `Chart`,
`BarChart`, `Tree`.** The set that draws data.

## Decisions in

T-004 and `specs/WIDGET_MODEL.md` §3. Settled.

**Open questions to settle:** Q-2 / O-W1 (the `Tree` widget's node store —
recommendation: a trait over the application's own store).

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| 0.12.0 | **`Gauge` and `LineGauge`** — a ratio, a label, partial-block fill | eighth-block partial fill correct at every ratio |
| 0.12.1 | **`Sparkline` and `BarChart`** — the eighth-block bars, horizontal and vertical | a series of every shape, including all-zero and single-point |
| 0.12.2 | **`Scrollbar`** — track, thumb, the four edges, the position arithmetic | thumb size and position exact at every content/viewport pair |
| 0.12.3 | **`Canvas`** — braille (2×4) and half-block (1×2) sub-cell drawing, points, lines, rectangles, the coordinate transform | Bresenham lines asserted against a reference, in integer arithmetic |
| 0.12.4 | **`Chart`** — axes, labels, datasets over `Canvas` | a chart at three sizes with labels that fit |
| 0.12.5 | **`Tree`** — the model trait, expansion state, indentation guides | a tree over a real directory listing |
| 0.12.6 | **Close** | `done/0.12/`, `0.13.0.md` written |

## Checklist

### 0.12.0 — gauges
- [ ] a ratio as a `Ratio(num, denom)` or a percentage — **never a float** (T-061's reasoning applies here too)
- [ ] partial fill with the eighth-block glyphs `U+2588`…`U+258F`, correct at every eighth
- [ ] a label centred in display width over the filled and unfilled regions, with the style flipping at the boundary
- [ ] `LineGauge` as the one-row variant
- [ ] ratios of exactly 0 and exactly 1, and a zero-width area

### 0.12.1 — sparkline and bar chart
- [ ] the eight vertical block glyphs; the eight horizontal ones for the horizontal bar chart
- [ ] a maximum taken from the data or supplied
- [ ] all-zero data, single-point data, data longer than the area, data shorter
- [ ] bar groups, labels, and value rendering for `BarChart`
- [ ] integer scaling with an explicit rounding rule, stated once

### 0.12.2 — `Scrollbar`
- [ ] the four edges, each with its own glyph set (track, thumb, and optional begin/end arrows)
- [ ] `ScrollbarState { content_length, position, viewport_content_length }`
- [ ] the thumb's size and position from integer arithmetic with a stated rounding rule, asserted exactly at every pair over a swept range — including a thumb that would round to zero and is clamped to one
- [ ] content shorter than the viewport (no thumb, or a full thumb — decided and stated)

### 0.12.3 — `Canvas`
- [ ] braille: `U+2800` + an eight-bit pattern, 2×4 sub-cells
- [ ] half-block: `U+2580`/`U+2584`/`U+2588`, 1×2 sub-cells, with the foreground/background trick for two colours per cell
- [ ] a coordinate transform from a caller's space to sub-cell space, in integer arithmetic
- [ ] points, lines (Bresenham), rectangles, and filled rectangles
- [ ] **the line algorithm asserted against a committed reference table**, because an off-by-one in Bresenham is invisible at a glance and permanent
- [ ] colour per cell, with the documented limitation that braille gives one colour per 2×4 block

### 0.12.4 — `Chart`
- [ ] x and y axes with bounds, labels and titles
- [ ] datasets with a name, a style, and a marker (dot, braille, block)
- [ ] a legend, placed and sized
- [ ] label collision handled by a stated rule, not by luck
- [ ] three sizes, including one where the labels do not fit

### 0.12.5 — `Tree`
- [ ] Q-2 decided and recorded
- [ ] the model interface: children of a node, a node's label, whether it has children
- [ ] expansion state owned by the application, keyed by an application-supplied identity
- [ ] indentation guides as a `LineSet`
- [ ] a tree over a real directory listing as the example, which is also the honest test

## Gate

The `Canvas` line reference table, the `Scrollbar` swept-pair assertion, and
every widget's golden set from 0.11's rule.

## Watch for

- **Sub-cell drawing is where rounding errors become visible.** Every scaling
  rule in this cycle is stated once, in a comment, and asserted — a "close
  enough" rounding is a chart whose last point is off the axis.
- **No floats.** A chart is the most tempting place in the library to reach for
  one, and the determinism claim does not survive it. Fixed-point (`tfp64`) is
  available and deterministic if a ratio genuinely needs fractional precision.
