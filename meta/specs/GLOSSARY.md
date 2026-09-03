# Glossary

One word per concept, one concept per word. Where the terminal world uses a
word two ways, this table says which one `ntui` means and what the other one is
called instead.

| Term | Means, in `ntui` |
|---|---|
| **cell** | one grid position on the screen, holding one grapheme cluster. Not "character", not "column". |
| **cluster** | a grapheme cluster (UAX #29). The unit a cell holds and the unit text is measured in. |
| **codepoint** | a Unicode scalar value. Used only inside the text layer; never a unit of the screen. |
| **width** | display columns a cluster occupies: 0, 1 or 2. Never a byte count and never a codepoint count. |
| **extent** | the size of something along one axis, in cells. Used in the layout engine so that "width" keeps its one meaning. |
| **surface** | a `Rect` plus a borrow of a buffer; what a widget draws into. |
| **buffer** | the cell grid. Two exist: **front** (believed on screen) and **back** (being composed). |
| **frame** | one composition and emission. |
| **damage** | the per-row span a write touched; an over-approximation of what changed. |
| **diff** | the comparison of back against front that decides what is emitted. |
| **pen** | the style currently in effect on the terminal, as the renderer models it. |
| **device** | `src/term/` — the descriptor, termios, size and signals. |
| **capability** | something the terminal can do, resolved once at startup and then a constant. |
| **probe** | the startup round trip that asks the terminal what it can do. |
| **calibration** | the startup round trip that measures how the terminal *renders width*. |
| **restore record** | the statically allocated structure `failsafe` reads to give the terminal back. |
| **mode** | a terminal state `ntui` turned on and must turn off; each has a bit in the restore record. |
| **the reactor** | the language's epoll-based readiness mechanism, reached through `io_watch` and `suspend_io`. |
| **the floor** | the language's runtime, `runtime/npkrt.ll`. |
| **the oracle** | the mini-VT in `tests/vt/` that parses what the renderer emitted back into a screen. |
| **golden** | a committed triple of (input program, expected bytes, expected screen). |
| **the budget** | the nine public error identities (`SAFETY.md` §3), and the rule that there are nine. |
| **arm** | one `pick` case in a consuming program's `failsafe`. |
| **event** | one item the runtime hands to `update`. |
| **command** | a request an application makes of the runtime through `Ctx`. Never a closure; the language has none. |
| **post** | a message sent into the runtime from a background task, through the eventfd. |
| **tick** | a timer or frame pulse, delivered as an event. |

## Words deliberately not used

| Not used | Because |
|---|---|
| "character" | ambiguous between byte, codepoint and cluster, which is the ambiguity this library exists to be careful about |
| "screen" as a noun for the buffer | the screen is the terminal's; the buffer is ours |
| "redraw" for the diff | a redraw is the decision to compose a frame; the diff is what gets emitted |
| "window" | `ntui` has no windows. A region is a `Rect`; a bordered region is a `Block`. |
| "component" | there is no component tree (T-004); there are widgets, which are values |
| "handler" | there are no callbacks; there is `update` |
| "flush" | the renderer emits and the device writes; there is no buffered state to flush |
| "attribute" for a colour | attributes are the boolean SGR flags; colours are colours |
