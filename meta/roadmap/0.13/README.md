# Cycle 0.13 — Input widgets

**`TextInput` and `TextArea`.** The two widgets that need the whole stack:
input decoding, grapheme segmentation, width, layout, and cursor placement.

## Why last of the widget cycles

Because they are the only widgets that consume events as well as produce cells,
and because a text editor is where every text-model mistake surfaces at once —
a cursor that moves by bytes, a selection that splits a cluster, a width that
disagrees with the terminal.

## Decisions in

`specs/WIDGET_MODEL.md` §3 and §7, `specs/TEXT_MODEL.md` §3. Settled.

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| 0.13.0 | **The text buffer** — a gap buffer or a line vector, with cluster-indexed positions | every position operation in clusters, never in bytes |
| 0.13.1 | **`TextInput`** — one line, cursor, selection, horizontal scroll | every editing operation, with mixed-width content |
| 0.13.2 | **Key handling** — the binding table, and how an application overrides it | a documented default binding set, and a program that replaces it |
| 0.13.3 | **`TextArea`** — many lines, wrapping, a viewport, vertical scroll | a 100 000-line document scrolled and edited |
| 0.13.4 | **Close** | `done/0.13/`, `0.14.0.md` written |

## Checklist

### 0.13.0 — the text buffer
- [ ] the representation chosen and recorded as a decision, with the measurement behind it
- [ ] **every position is a cluster index**, and the byte offset is derived (T-020)
- [ ] insert, delete, and replace at a cluster boundary; never inside one
- [ ] a position operation on an empty buffer, at the start, and at the end
- [ ] undo/redo, or an explicit statement that it is the application's — decided, not left open

### 0.13.1 — `TextInput`
- [ ] cursor movement by cluster, by word, to line start and end
- [ ] insert, backspace, delete, delete word
- [ ] selection with shift-movement, and replace-selection on insert
- [ ] horizontal scroll keeping the cursor visible, computed in display width
- [ ] a placeholder, a mask (for passwords), and a maximum length
- [ ] paste inserted as one operation from an `Event.Paste`, **not** as synthetic keys (T-035)
- [ ] the cursor request set through `Ctx` at frame close (R-19), so the terminal's real cursor is where the user's is
- [ ] mixed-width content: a cursor after a CJK character is two columns along, and a golden case says so

### 0.13.2 — key handling
- [ ] a default binding table, documented, covering the readline-ish set most users expect
- [ ] `key_matches()` (O-I1) used for every binding, so the protocol asymmetry does not leak into applications
- [ ] the table replaceable by the application — as data, not as callbacks (there are none)
- [ ] a program that replaces the whole table, as the test that the mechanism works

### 0.13.3 — `TextArea`
- [ ] many lines with a vertical viewport
- [ ] wrapping shared with `Paragraph` — one implementation, not two
- [ ] cursor movement across wrapped lines behaving as the user expects (down moves a visual row, not a logical line), stated and tested
- [ ] a 100 000-line document: scroll to top, bottom, and a random position; edit at each
- [ ] line numbers as an option

## Gate

A working editor-shaped example over a mixed-script document, with a golden
case per editing operation.

## Watch for

- **Byte indices are the bug.** Every position in this cycle is a cluster
  index. The moment one is a byte offset, an emoji breaks the cursor.
- **`in` and `end` are keywords**, and a text buffer wants both constantly.
- **This cycle is where 0.2's segmenter earns its keep** — or where its holes
  appear. A failure here is a finding about 0.2, and it goes back there rather
  than being worked around.
