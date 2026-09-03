# Widgets

---

## 1. What a widget is

**Rule W-1.** A widget is a **plain value** that knows how to draw itself into a
rectangle. It is not an object, it has no identity, it holds no reference to a
parent, and it does not exist between frames. An application constructs one,
draws with it, and drops it — usually all in one statement.

```nitpick
Paragraph:p = paragraph("hello")
    .align(Align.Center)
    .wrap(Wrap.Word)
    .style(st);
drop p.render(buf, area);
```

**Rule W-2 — the trait.**

```nitpick
pub trait:Widget = {
    func:render = NIL(Self->:self, Buffer->:buf, Rect:area);
};
```

`Self->` because a widget may own a string and a by-value receiver would
consume it per call (D-183, the same reason `IO_REFERENCE.md`'s stream traits
take `Self->`). Not `async`: drawing does not wait.

**Rule W-3 — a widget draws only inside `area`.** Every write goes through the
clipping accessors (`SCREEN_MODEL.md` §9), so a widget that computes one past
its edge is clipped rather than corrupting a neighbour. This is checked, not
trusted: the golden suite renders every widget into a rect surrounded by a
sentinel pattern and asserts the sentinel is untouched.

---

## 2. Builders, without closures and without inheritance

**Rule W-4.** Widgets are configured by **chained methods that return `Self`**,
which the language supports directly through inherent impls and UFCS. There is
no builder type, no `Default` derive (D-123 removed it, and for good reasons —
a default that carries meaning is a value nobody chose), and no options struct
with twenty fields.

Every widget has a bare constructor function naming its required content —
`paragraph(text)`, `block()`, `list(items)` — because **the language has no
static methods** (D-185: every trait and impl method takes `self`), so
`Paragraph.new(…)` is unspellable. The ecosystem's convention for this is the
bare function, and `ntui` follows it.

**Rule W-5 — a chained setter consumes and returns.** `move Self` in, `Self`
out. Owning fields make the alternative (a `Self->` setter returning nothing)
awkward at the call site and this shape reads the way people expect. It costs a
struct copy per call, on a value that exists for one statement.

---

## 3. The set, at 1.0

**Containers and decoration**

| Widget | What it does |
|---|---|
| `Block` | a border, a title, and an inner area; the thing everything else is drawn inside |
| `Clear` | blanks a region — the primitive a popup is built from |
| `Padding` | shrinks an area; a `Rect` helper promoted to a widget for chaining |

**Text**

| Widget | What it does |
|---|---|
| `Line` | a run of styled `Span`s on one row, with an alignment |
| `Paragraph` | many `Line`s, wrapped or not, scrolled by an offset |
| `Text` | the owned multi-line value `Paragraph` renders |

**Data**

| Widget | What it does |
|---|---|
| `List` | selectable rows with a highlight, a symbol, and a scroll offset |
| `Table` | columns with per-column `Constraint`s, a header, and row selection |
| `Tree` | a collapsible hierarchy over an `arena<T>`-held model |

**Indicators**

| Widget | What it does |
|---|---|
| `Gauge` | a proportional bar, with a ratio and a label |
| `LineGauge` | a one-row variant |
| `Sparkline` | a bar chart of one series, using the eighth-block glyphs |
| `Scrollbar` | a track and a thumb, on any of the four edges |
| `Tabs` | a one-row selector |

**Graphics**

| Widget | What it does |
|---|---|
| `Canvas` | a sub-cell drawing surface over braille (2×4) or half-blocks (1×2), with points, lines, rectangles and a coordinate transform |
| `Chart` | axes, labels and datasets over a `Canvas` |
| `BarChart` | categorical bars, horizontal or vertical |

**Input**

| Widget | What it does |
|---|---|
| `TextInput` | one line, with a cursor, selection, and horizontal scroll |
| `TextArea` | many lines, with wrapping and a viewport |

**Rule W-6.** Nothing in this table is a container that owns children. There is
no `VBox`, no `Stack`, no `Grid`. Composition is `layout_split` followed by
drawing into the returned rects, which is one line more at the call site and
removes an entire category of "why is my widget not visible" from the library.

---

## 4. Stateful widgets

**Rule W-7.** A widget that needs memory across frames — a `List`'s scroll
offset and selection, a `Table`'s, a `Scrollbar`'s position, a `TextInput`'s
cursor — takes it as an **explicit state value the application owns**:

```nitpick
pub struct:ListState = { uint32:offset; uint32?:selected; };

drop list(items).render_stateful(buf, area, @st.list_state);
```

The application declares the state in its own struct, passes a borrow, and the
widget reads and updates it. Nothing is hidden, the state survives because the
application holds it, and a test constructs the state directly.

**Rule W-8.** A stateful widget's state is a plain POD value and its fields are
public. It is data the application is expected to read (which row is selected
is the application's question) and set (jumping to a row is the application's
action). A getter/setter pair over a `uint32` would be ceremony.

---

## 5. Text layout

**Rule W-9 — the wrap rule, stated.** `Wrap.None` clips at the area's edge.
`Wrap.Word` breaks at the last space-class boundary that fits; if a single run
between boundaries is wider than the area, it breaks at the last **grapheme
cluster** that fits. `Wrap.Char` always breaks at the last cluster that fits.

"Space class" is `U+0020`, `U+0009`, and the Unicode `Zs` category. This is
**not** UAX #14 line breaking (`TEXT_MODEL.md` §3 says why it is absent), and
the documentation says so plainly rather than implying a Unicode conformance
the library does not claim.

**Rule W-10 — control characters in widget text are handled, not refused.**
`\n` starts a new `Line`. `\t` advances to the next multiple of `tab_width`
(default 8, per widget). `\r` is dropped. Every other C0 and C1 character
renders as its Unicode control picture (`U+2400 + n` for C0) when the terminal
can, else as `?`. A widget never passes a control byte to the renderer, which
is what `SCREEN_MODEL.md` §2's "controls never reach a cell" means in practice.

**Rule W-11 — alignment is computed in display width, not in bytes or
codepoints.** Centring a line containing an emoji and a CJK character is the
test case, and it is in the golden suite.

---

## 6. Borders

**Rule W-12.** `Block`'s border set is a value, not a flag: `LineSet` names
twelve glyphs (four corners, four edges, four tees plus a cross), and the
library ships `PLAIN`, `ROUNDED`, `DOUBLE`, `THICK`, `QUADRANT_INSIDE`,
`QUADRANT_OUTSIDE` and `ASCII`. A terminal or font without box-drawing
characters gets `ASCII` by the application's choice, not by a capability
guess — `ntui` cannot know what the font has.

**Rule W-13.** Borders are drawn with the **join glyphs** where two blocks
share an edge only if the application draws them that way; the library does not
detect adjacency. Automatic border joining requires knowing what is already in
the cell, which means reading the back buffer, which is a coupling that pays
for itself only in a layout system with a container tree — and W-6 declined
one.

---

## 7. What a widget may not do

**Rule W-14.** A widget may not: read the front buffer, write outside its area,
retain a borrow past `render`, allocate proportional to the *screen* (only to
its own content), await anything, or consult the environment or the clock. A
widget is a function from a value and a rect onto cells.

**Rule W-15.** The clock exclusion is deliberate and is the one that surprises:
an animated spinner does not tick itself. It takes a frame counter or an
elapsed `Duration` from the application, which got it from a timer
(`EVENT_MODEL.md` §5). That is what makes a frame reproducible — a golden test
of a spinner is a test at a stated frame number.

---

## 8. Open items

- **O-W1 — the `Tree` widget's model.** A collapsible tree needs a node store,
  and `arena<T>` + `Handle<T>` is what the language provides for graph-shaped
  data. Whether `ntui` ships the store or takes a trait the application
  implements over its own store is open; recommendation is the trait, so a
  file browser over a real directory tree does not have to mirror it into an
  arena first.
- **O-W2 — image protocols** (Kitty graphics, Sixel, iTerm2 inline). Out of
  scope at 1.0, recorded as wanted, and `Caps.sixel` is already carried so the
  detection is not re-litigated later. It is its own cycle: the protocols
  disagree about placement, scrolling and lifetime, and the cell model has to
  learn about a region it does not own.
- **O-W3 — accessibility.** A screen reader on a TUI reads the terminal, not
  the program, so the library's contribution is to emit text in a sensible
  order and to keep the cursor where the focus is. Both are properties the
  golden oracle could assert. Not planned for 1.0; recorded so it is a decision
  rather than an oversight.
