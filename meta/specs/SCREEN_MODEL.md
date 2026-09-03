# The screen: cells, buffers, and the renderer

---

## 1. The determinism rule

**Rule R-1.** The bytes a frame emits are a **pure function of
`(front, back, caps, cursor)`**. The renderer reads no environment, makes no
syscall, consults no clock, and never reads from the terminal. It returns a
`Bytes`; the device writes it (`TERMINAL_MODEL.md` §9).

Everything else in this document exists to keep that true, and it is what makes
the golden suite (`TESTING.md` §5) possible: a test constructs two buffers,
calls the renderer, and asserts on the bytes.

---

## 2. `Cell`

```nitpick
pub struct:Cell = {
    uint8[14]:text;    // the grapheme cluster's UTF-8, when it fits
    uint8:len;         // 0..14 = inline length; 255 = spilled, see below
    uint8:flags;       // §2.1
    Style:style;       // 16 bytes, STYLE_MODEL.md
};                     // 32 bytes, align 4
```

**Rule R-2 — 14 bytes inline, spill beyond.** Fourteen holds every cluster that
occurs in practice: a base plus three combining marks, a flag pair, a
base-plus-VS16, a keycap sequence. It does not hold a long emoji ZWJ family
(seven codepoints, up to 25 bytes), so `len == 255` means the first four bytes
of `text` are a `uint32` index into the owning buffer's **cluster pool**, and
the cluster's bytes are there.

The number is a measurement, not a guess, and the plan's step that sets it
measures a corpus first (cycle 0.6.0). What is *not* negotiable is that `Cell`
has no owning field: a `string` here would make the cell move-only (TYPE-046)
and a grid of move-only cells cannot be copied, cleared, or diffed.

**Rule R-3 — `len == 0` is a blank cell** and its `text` is ignored. A blank is
not a space: it is "nothing has been written here", it renders as a space, and
it compares equal to another blank with the same style regardless of `text`'s
contents. The comparison function is the authority and it is one function.

### 2.1 `flags`

| Bit | Name | Meaning |
|---|---|---|
| 0 | `CELL_WIDE_LEAD` | this cell holds a width-2 cluster; the next cell is its tail |
| 1 | `CELL_WIDE_TAIL` | this cell is the second half of the cell to its left and holds no text of its own |
| 2 | `CELL_SKIP` | the renderer must not emit this cell (it is covered by a tail, or clipped) |

**Rule R-4 — wide characters occupy two cells, explicitly.** A width-2 cluster
is written into cell *n* with `CELL_WIDE_LEAD` and cell *n+1* is overwritten
with a `CELL_WIDE_TAIL` blank carrying the same style. Nothing anywhere infers
width from a cell's contents at render time; the width question is answered
once, when the text is written, by `TEXT_MODEL.md` §4.

**Rule R-5 — writing over half of a wide pair blanks the other half.** Writing
a cluster into a `CELL_WIDE_TAIL` cell turns its lead into a blank of the same
style; writing into a `CELL_WIDE_LEAD` blanks its tail. Otherwise the buffer
holds a state the terminal cannot represent, and the shear that follows shows
up three widgets away from the write that caused it.

**Rule R-6 — a width-2 cluster does not fit in the last column.** It is
replaced by a blank (`TEXT_MODEL.md` X-13). Autowrap is off
(`TERMINAL_MODEL.md` §7) so nothing scrolls, but half a glyph is still wrong.

---

## 3. `Buffer`

```nitpick
pub struct:Buffer = {
    Vec<Cell>:cells;        // rows * cols, row-major
    uint16:rows;
    uint16:cols;
    Bytes:pool;             // the cluster pool for spilled cells
    Vec<LinkEntry>:links;   // id -> URI; index 0 is reserved as "no link"
    Vec<RowDamage>:damage;  // one per row, §5
};
```

**Rule R-7 — indexing goes through one accessor pair.**
`buffer_at(b, row, col) -> Cell` and `buffer_set(b, row, col, cell)`, and
nothing outside `screen/buffer.npk` touches `cells.items`. That is what makes
the bound one obligation for cycle 1.5 instead of hundreds
(`SAFETY.md` S-17, `VERIFICATION.md` §3).

**Rule R-8 — a resize reallocates and clears.** It does not attempt to preserve
content: the application repaints on `Resize` anyway, preserving would need a
policy for what happens to a widget that is now off-screen, and the policy
would be wrong for somebody. Both buffers are resized together and the whole
screen is marked damaged.

**Rule R-9 — the pool and the link table are per-buffer and are cleared with
it.** They grow monotonically within a frame and reset at `buffer_clear`, which
is what the back buffer gets at the start of every frame. No reference counting
and no reuse across frames: a frame is a fresh composition and the pool is
scratch.

---

## 4. The two buffers

**Rule R-10.** The `Screen` owns a **front** buffer (what the terminal is
believed to show) and a **back** buffer (what the next frame should show).
Widgets draw into the back buffer. `screen_render` diffs back against front,
emits, and then **swaps** — the back becomes the front and the new back is
cleared.

Swap rather than copy: a copy of a 200×60 grid is 384 KB per frame and a swap
is two pointer writes.

**Rule R-11 — the front buffer is invalidated, not guessed, when the terminal
may have changed underneath.** `screen_invalidate()` marks every cell damaged
and is called on resize, on resume from suspend, and by an application that
knows something else wrote to the terminal. There is no attempt to detect it.

---

## 5. Damage

```nitpick
pub struct:RowDamage = { uint16:lo; uint16:hi; };   // hi < lo means "clean"
```

**Rule R-12.** Damage is tracked **per row as one span**, widened by every
write. Not per cell (a bitmap costs a branch per write and the renderer would
still coalesce), and not per screen (which makes every frame a full repaint).

**Rule R-13.** A write marks damage even when it writes the same value. The
renderer's diff decides what actually changed; damage is a cheap
over-approximation that bounds where the diff has to look. Over-approximating
is the safe direction — the diff catches the rest — and a write path that
compares before marking pays the comparison twice.

---

## 6. The renderer

`screen_render(front, back, caps, cursor) -> Bytes`. In order:

1. **Frame open.** `CSI ? 2026 h` when `caps.synchronized` (§8), then
   `CSI ? 25 l` if the cursor is currently shown, then `CSI 0 m` and the pen
   reset (`STYLE_MODEL.md` Y-13).
2. **For each row with damage**, and no others.
3. **Within the row**, compute the changed cells between `lo` and `hi`
   inclusive by comparing `back` against `front`, then split them into **runs**
   separated by gaps of at least `NTUI_GAP` unchanged cells.
4. **For each run**: position the cursor once (§7), then emit its cells left to
   right, emitting a style transition before any cell whose style differs from
   the pen.
5. **Trailing-blank optimisation**: if a run reaches the last column and every
   cell from some position `p` onward is blank with the pen's current
   background, and `hi - p >= NTUI_EL_MIN`, emit `CSI K` at `p` instead of the
   spaces.
6. **Frame close.** Position the cursor where the application asked, show it if
   the application asked, then `CSI ? 2026 l` when synchronized.

**Rule R-14 — the constants are fixed and named**, because a heuristic that
varies is a rendering that varies:

| Constant | Value | Why |
|---|---|---|
| `NTUI_GAP` | 8 | a cursor-position sequence is 6–10 bytes; below 8 unchanged cells it is cheaper to re-emit them |
| `NTUI_EL_MIN` | 4 | `CSI K` is 3 bytes |

Changing either is a recorded decision and re-records every golden file, which
is exactly the friction that should exist around a change to what the library
emits.

**Rule R-15 — no cell is emitted twice and no `CELL_WIDE_TAIL` is emitted at
all.** Emitting the lead advances the terminal's cursor by two, so the tail is
skipped; the renderer's position tracking accounts for it. A test asserts that
the emitted stream's implied cursor track matches the cells it wrote, which is
the invariant this rule is really about.

**Rule R-16 — the renderer emits no character it did not get from a cell.** No
padding spaces to "reach" a position (that is what cursor positioning is for),
no clearing that was not asked for, and no newline ever: `\n` at the last row
scrolls, and the renderer never scrolls.

---

## 7. Cursor positioning

**Rule R-17.** Absolute positioning is `CSI <row+1> ; <col+1> H`. The renderer
uses the shorter relative forms when they are shorter **and unambiguous**:

| Situation | Emitted |
|---|---|
| same row, `n` columns right, `n <= 4` | `CSI <n> C`, or `n` × the cheaper of that and re-emitting |
| column 0 of the current row | `CR` (`\r`), one byte |
| next row, column 0 | `CR` then `CSI B` — **never** `\n`, which scrolls at the bottom |
| anything else | the absolute form |

**Rule R-18 — the renderer's model of the cursor is exact.** It knows the
column after every cell it emits, because it knows each cell's width. It does
**not** rely on autowrap, which is off. If its model and the terminal's ever
disagree the frame is wrong, so the mini-VT oracle (`TESTING.md` §5) checks
exactly this on every golden test.

**Rule R-19 — the application's cursor is set at frame close, once.** A widget
that wants the cursor (a text input) records a request; the last request in the
frame wins, and "no request" means the cursor stays hidden. This is one
mechanism rather than widgets moving a shared cursor as a side effect of
drawing.

---

## 8. Synchronized output

**Rule R-20.** When `caps.synchronized`, **every** frame is wrapped in
`CSI ? 2026 h` … `CSI ? 2026 l`, unconditionally and at exactly one place. Not
per widget, not per region, not conditional on frame size. A terminal that
supports it renders the frame atomically and the tearing that a partially
written frame produces disappears; a terminal that does not ignores both.

**Rule R-21.** A frame that emits **nothing** — no damage, or damage that the
diff resolves to no changes — emits *nothing at all*, including the
synchronization wrapper and including the pen reset. An idle application writes
zero bytes to the terminal, which is what makes it free to run at 60 frames a
second when nothing is happening.

---

## 9. Drawing surfaces

**Rule R-22.** Widgets do not receive the `Buffer`. They receive a **`Surface`**
— a `Rect` plus a borrow of the buffer — whose coordinates are local to the
rect and whose writes are clipped to it:

```nitpick
pub struct:Surface = { Buffer->:buf; Rect:area; };
```

A write outside the surface is **silently clipped**, not an error. A widget
computing a position one past its area is an everyday occurrence — a border, a
scrollbar, a shadow — and making it an error means every widget carries bounds
checks that duplicate the one the surface already has.

**Rule R-23.** `Surface` is a **borrow** and therefore second class (D-004): it
cannot be returned, stored in a struct that outlives the call, sent through a
channel, or held across an `await`. A widget's `render` takes one and does not
keep it. This is enforced by the compiler, not by convention, and it is why the
widget trait's method is shaped the way `WIDGET_MODEL.md` §2 shapes it.

**Rule R-24 — the surface API is the whole drawing vocabulary**, and it is
small:

```
surface_set_cluster(s, col, row, bytes, style) -> uint16   // returns width consumed
surface_set_str(s, col, row, text, style)      -> uint16   // returns columns consumed
surface_fill(s, area, cluster, style)
surface_clear(s, area, style)
surface_set_style(s, area, patch)              // restyle without touching text
surface_hline / surface_vline(s, …, LineSet, style)
surface_sub(s, area) -> Surface                // a clipped child, coordinates rebased
```

Everything a widget draws is one of these. There is no "draw a rectangle" and
no "draw text with wrapping": those are widgets (`WIDGET_MODEL.md`), built from
these, and keeping the primitive set this small is what makes the renderer's
invariants checkable.

---

## 10. Deferred: scroll optimisation

**Rule R-25.** The renderer does **not** use `CSI S` / `CSI T` (scroll up/down)
or `CSI L` / `CSI M` (insert/delete line) in this version, and this is a
decision rather than an omission.

They are a large win for a scrolling list — one sequence instead of repainting
every row — and they are the single most common source of rendering bugs in
mature TUI libraries, because they require the renderer to model the terminal's
scrollback, its margins and its clearing behaviour, all of which vary. The
front buffer would also have to be shifted to match, which turns a pure diff
into a stateful transformation.

They belong in a cycle of their own, after the golden oracle can prove a scroll
produced the same screen as a repaint. The plan has that cycle
(`ROADMAP.md`, 0.13) and it is gated on exactly that ability.
