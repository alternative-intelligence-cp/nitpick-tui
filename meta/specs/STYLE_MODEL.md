# Style: colour and attributes

---

## 1. `Style`

```nitpick
pub struct:Style = {
    Color:fg;          // 4 bytes
    Color:bg;          // 4 bytes
    Color:ul;          // 4 bytes — the underline colour (SGR 58)
    uint16:attrs;      // §3
    uint16:link;       // hyperlink id into the buffer's link table, 0 = none
};                     // 16 bytes, align 4
```

**Rule Y-1.** `Style` is a plain 16-byte value with **no owning fields**, so a
cell grid is an array of `Cell` and a copy is a copy. This is a hard
requirement of the language rather than an optimisation: an owning field would
make `Cell` move-only (TYPE-046) and a grid of move-only values unusable.

**Rule Y-2.** A hyperlink URI is too large for a cell, so a `Style` carries a
**link id** and the URI lives in the buffer's link table (`SCREEN_MODEL.md`
§3). Id 0 is "no link" and is never a valid entry.

---

## 2. `Color`

```nitpick
pub struct:Color = { uint8:kind; uint8:a; uint8:b; uint8:c; };
pub enum:ColorKind = { Default; Ansi; Indexed; Rgb; };
```

| `kind` | Meaning | Fields |
|---|---|---|
| `Default` | the terminal's own foreground/background | all zero |
| `Ansi` | one of the 16 named colours, `a` in 0…15 | `a` |
| `Indexed` | the 256-colour cube, `a` in 0…255 | `a` |
| `Rgb` | 24-bit, `a`/`b`/`c` = R/G/B | all three |

Constructors: `color_default()`, `color_ansi(n)`, `color_indexed(n)`,
`color_rgb(r, g, b)`, plus the sixteen named ones (`color_red()`,
`color_bright_red()`, …).

**Rule Y-3 — `Default` is not black and is not white.** It is *the terminal's
own*, and it is a distinct case because a program that paints its background
with an explicit colour looks wrong in a terminal with a themed or transparent
background. Widgets default to `Default` for both, and an application opts in
to painting.

**Rule Y-4 — `Ansi` and `Indexed` are separate kinds even though the first 16
indices coincide.** `Ansi(1)` emits `SGR 31`, which the terminal renders with
its *configured* red and which honours the bold-is-bright convention;
`Indexed(1)` emits `SGR 38;5;1`, which some terminals render from the palette
and some from a fixed table. They are different requests and a library that
folds them cannot express the first.

---

## 3. Attributes

`attrs` is a `uint16`. Bits 0…9 are boolean attributes; bits 12…14 hold the
underline style. (A library cannot declare a flag family — D-230 makes
`TY_FLAGS` four compiler-known families — so this is a plain integer with named
constants and predicates, exactly as `Mods` is in `INPUT_MODEL.md` §3.)

| Bit | Constant | SGR on | SGR off |
|---|---|---|---|
| 0 | `ATTR_BOLD` | `1` | `22` |
| 1 | `ATTR_DIM` | `2` | `22` |
| 2 | `ATTR_ITALIC` | `3` | `23` |
| 3 | `ATTR_BLINK` | `5` | `25` |
| 4 | `ATTR_REVERSE` | `7` | `27` |
| 5 | `ATTR_HIDDEN` | `8` | `28` |
| 6 | `ATTR_STRIKE` | `9` | `29` |
| 7 | `ATTR_OVERLINE` | `53` | `55` |
| 8 | `ATTR_RAPID_BLINK` | `6` | `25` |
| 9 | `ATTR_FRAKTUR` | `20` | `23` |

| Bits 12–14 | Underline style | SGR |
|---|---|---|
| 0 | none | `24` |
| 1 | single | `4` |
| 2 | double | `4:2` |
| 3 | curly | `4:3` |
| 4 | dotted | `4:4` |
| 5 | dashed | `4:5` |

**Rule Y-5 — `22` turns off both bold and dim**, and this is the reason §5's
emission rule is what it is. There is no SGR that turns off bold while leaving
dim, so a renderer that emits individual off-codes cannot express every
transition. Nobody discovers this from the spec; everybody discovers it from a
screen where the dim text went bright.

**Rule Y-6.** `ATTR_RAPID_BLINK` and `ATTR_FRAKTUR` exist for completeness and
are dropped on almost every terminal. They are in the table so that a `Style`
round-trips through the emitter and the mini-VT oracle without a special case,
not because anybody should use them.

---

## 4. Degradation

**Rule Y-7.** Degradation is a **pure, total function** of `(Color, ColorDepth)`
applied once, at emission, in the renderer (`CAPABILITIES.md` §13). A `Style`
value always means what its author wrote.

**RGB → Indexed256.** The standard xterm cube. For each channel, the six cube
levels are `0, 95, 135, 175, 215, 255`; the greyscale ramp is
`8 + 10*i` for `i` in 0…23. `ntui` computes **both** candidates — nearest cube
cell and nearest grey — and picks the one with the smaller squared Euclidean
distance in RGB, breaking a tie toward the cube. This is the mapping every
serious implementation converged on, and stating the tie-break is what makes
the result reproducible.

**Indexed256 → Ansi16.** A fixed 256-entry table, generated into
`src/style/degrade.npk`, mapping each index to the nearest of the sixteen at
the *standard* xterm RGB values for those sixteen. It is a table rather than a
computation because the sixteen are terminal-configurable and any computation
would be pretending to know their real values.

**RGB → Ansi16** composes the two, and is a separate table so it does not lose
twice.

**Anything → None.** Colour is dropped; attributes survive.

**Rule Y-8 — an unsupported attribute is dropped, never substituted**
(`CAPABILITIES.md` C-13). A curly underline on a terminal without styled
underlines becomes a single underline (the nearest *weaker* form of the same
thing); a single underline on a terminal without underline becomes nothing. It
never becomes reverse video.

**Rule Y-9.** The degradation tables have a test that walks all 2^24 RGB values
against the cube mapping's invariants — that the result is in range, that it is
idempotent on exact cube colours, and that the grey ramp is chosen for exact
greys — because a table with one wrong entry is invisible until somebody
screenshots it.

---

## 5. Emission

**Rule Y-10 — the renderer tracks a pen** (the style currently in effect on the
terminal) and emits the transition, never the whole style.

**Rule Y-11 — the transition rule, stated so it is deterministic.**

> If the target's attribute set is a superset of the pen's and neither
> colour needs to become `Default`, emit only the additions: the new attribute
> codes, and an SGR colour for each colour that changed.
>
> **Otherwise emit `CSI 0 m` and then the target in full.**

That covers Y-5's problem (any attribute that must be *removed* forces a
reset), covers the `Default` colour case (`39`/`49` exist but are unreliable on
some older terminals and a reset is one code shorter than two), and has no
branch a reader has to reason about. It costs a few bytes on transitions that
remove an attribute, which is not the common case.

**Rule Y-12 — one SGR per cell run, and the colon form for what needs it.**
Codes are joined into one `CSI …;…;… m`. Colours use the **colon**
sub-parameter form where the terminal supports it (`38:2::R:G:B`) and the
semicolon form otherwise (`38;2;R;G;B`), because the colon form is what
ECMA-48 actually specifies and the semicolon form is xterm's ambiguity — but
the semicolon form is what more terminals parse. The choice is a capability,
resolved once, and both are emitted by the same function from one table.

**Rule Y-13 — the pen is reset at the start of every frame.** A frame begins
with `CSI 0 m` and the pen set to the default `Style`. This costs four bytes
per frame and makes a frame's byte stream independent of the frame before it,
which is what makes a golden test a test of one frame rather than of a
history.

**Rule Y-14 — hyperlinks.** `OSC 8 ; ; <uri> ST` opens, `OSC 8 ; ; ST` closes.
The renderer opens when the pen's link id changes to non-zero, closes when it
changes to zero or to a different id (close-then-open). `Caps.hyperlinks` off
drops both and keeps the text. The `id=` parameter is not used: it exists to
join a link split across lines, and `ntui` emits contiguous runs so it has
nothing to join.

---

## 6. Composition

**Rule Y-15.** `style_patch(base, over)` produces a style where each field of
`over` that is *set* replaces `base`'s. "Set" needs a representation for
"unspecified", and rather than a parallel mask `ntui` uses a second type:

```nitpick
pub struct:StylePatch = {
    Color:fg; Color:bg; Color:ul;
    uint16:add;      // attributes to turn on
    uint16:remove;   // attributes to turn off
    uint16:link;
    uint8:set;       // which of fg/bg/ul/link this patch specifies
};
```

Two types rather than one with sentinel values, because a sentinel for
"unspecified colour" would collide with `Default`, which is a real colour
request and the most common one. This is the same reasoning D-069 applied to
`Result`'s removed `is_error` field: one fact, one representation, no pair that
can contradict.

**Rule Y-16.** Patching is associative and has an identity, and there is a test
that says so over a generated corpus. A theme system is patches all the way
down and a non-associative patch produces a theme whose result depends on the
order somebody happened to write the layers in.
