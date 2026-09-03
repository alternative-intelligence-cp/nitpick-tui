# Text: UTF-8, grapheme clusters, and display width

The single largest source of visible corruption in terminal programs is
disagreement about how many columns a piece of text occupies. Get it wrong by
one and every subsequent cell on the line is in the wrong place; get it wrong
inside a box and the box tears. This document says exactly what `ntui`
believes, where the belief comes from, and what it does when the terminal
believes something else.

---

## 1. The three questions, kept apart

They are routinely conflated and they are different:

| Question | Answer's unit | Where |
|---|---|---|
| How is this byte sequence decoded? | codepoints (`char32`) | §2 |
| Where can a string be split without breaking a character a user sees? | grapheme clusters | §3 |
| How many terminal columns does this occupy? | cells | §4 |

**Rule X-1.** The unit of the screen is the **grapheme cluster** (T-020). Not
the byte — `é` is two bytes. Not the codepoint — `é` may be two codepoints, and
a flag is two, and a family emoji is seven. A cell holds one cluster, and every
API that talks about a position on the screen talks in cells.

---

## 2. UTF-8

**Rule X-2.** Decoding is a table-driven DFA over bytes, total, with no
lookahead beyond the sequence being decoded and no allocation. It accepts
exactly the well-formed sequences of Unicode Table 3-7: no overlongs, no
surrogates (U+D800…U+DFFF), nothing above U+10FFFF, and the byte ranges
constrained per lead byte rather than by a general "continuation byte" rule.

**Rule X-3 — invalid input becomes U+FFFD by the maximal-subpart rule**
(T-023). One replacement character per maximal subpart of an ill-formed
sequence, which is the Unicode recommended practice and what browsers do.
Concretely: `E1 80 41` yields U+FFFD then `A`, not two U+FFFDs and not one.

Bytes are never dropped silently and never trusted. A terminal is an untrusted
input device — a paste can carry anything, and a program that decodes a paste
into a buffer it then indexes has to be right about length.

**Rule X-4.** `ntui`'s public text APIs take `string`, which the language
guarantees nothing about beyond being bytes. Where validity is required (a
label to be measured and drawn), the function validates and fails
`ETuiEncoding`; where it is not (a byte pass-through), it does not look.
**Which of the two a function is, is stated in its documentation comment**, and
the conformance suite feeds every public text entry point a corpus of
ill-formed input and requires either a clean `ETuiEncoding` or a defined
substitution — never a trap and never a silent smear.

**Rule X-5.** Encoding back to UTF-8 is exact and allocation-free into a caller
supplied `uint8[4]`, returning the length. The renderer needs this on the hot
path and must not be building strings.

---

## 3. Grapheme clusters

**Rule X-6.** Cluster boundaries are **UAX #29 extended grapheme clusters**,
implemented as the standard rule set over the `Grapheme_Cluster_Break` property
plus `Extended_Pictographic`, plus the Indic conjunct-break rules
(`InCB=Linker` / `InCB=Consonant`) introduced at Unicode 15.1. The rules
implemented, by their standard names: GB1–GB5, GB6–GB8 (Hangul), GB9, GB9a,
GB9b, GB9c (Indic conjunct), GB11 (emoji ZWJ), GB12/GB13 (regional indicator
pairs), GB999.

**Rule X-7.** The segmenter is an **incremental** function over a byte slice:
`(state, bytes) → (cluster_end, state)`. It is used in three places with three
different feeds — measuring a label, laying out wrapped text, and decoding a
paste — and one implementation with a carried state is what keeps those three
from disagreeing.

**Rule X-8.** The regional-indicator rule (GB12/GB13) is the one that needs
carried state rather than a two-codepoint window: whether an RI is the first or
second of a pair depends on how many RIs precede it in the *run*. The state
word carries the parity. This is called out because it is the rule every
hand-rolled segmenter gets wrong.

**Rule X-9.** `ntui` implements the **grapheme** boundary rules only. Word
(UAX #29 §4) and line (UAX #14) breaking are **not** in the library at this
version: word breaking is needed by a text editor, line breaking by justified
prose, and neither is needed by a widget toolkit whose wrap rule is stated in
[`WIDGET_MODEL.md`](WIDGET_MODEL.md) §5 as "break at the last space that fits,
else break the cluster run". Adding either is a recorded decision and its own
cycle, not a quiet expansion of the tables.

---

## 4. Display width

**Rule X-10 (T-022) — the algorithm, exactly, in order.** For a grapheme cluster,
width is decided by its **first** codepoint except where a later codepoint
promotes it:

1. If the first codepoint is a C0 or C1 control (`U+0000…U+001F`,
   `U+007F…U+009F`): **refused**. Controls never reach a cell; the layer above
   has already turned `\n`, `\t` and the rest into layout (WIDGET_MODEL.md §5),
   and anything else is rejected with `ETuiEncoding` or substituted with a
   visible replacement at the caller's choice.
2. `U+0000` alone, if it survives to here, is width 0 and renders as a space.
3. If the cluster contains `U+FE0F` (VARIATION SELECTOR-16) and its base is
   `Emoji`: **2**.
4. If the cluster contains `U+FE0E` (VARIATION SELECTOR-15): **1**.
5. If the base is `Emoji_Presentation=Yes`: **2**.
6. If the base is `East_Asian_Width` `W` or `F`: **2**.
7. If the base has `General_Category` `Mn`, `Me` or `Cf`, or is
   `Default_Ignorable_Code_Point`: **0**.
8. Otherwise: **1**.

Combining marks after the base add nothing — they are the same cluster and rule
7 already gives them zero on their own account.

**Rule X-11 — `East_Asian_Width=Ambiguous` is width 1**, and it is a
capability, not a constant: `Caps.ambiguous_wide` promotes the whole `A` class
to 2. It is off by default because the default is right for Latin locales and
for every modern terminal's default configuration, and it exists at all because
it is wrong in a CJK locale on a terminal configured for double-width ambiguous
characters, and being wrong there means every box in the program tears.

**Rule X-12 — ZWJ sequences are the known-unsolvable case, and are handled by
measurement, not by belief.** An emoji ZWJ sequence such as
`U+1F468 U+200D U+1F469 U+200D U+1F467` renders as one double-width glyph on a
terminal that composes it and as three separate double-width glyphs on one that
does not. **No static table can answer this**, because the answer is a property
of the font and the terminal, not of the text.

`ntui`'s answer is the only one that works:

- **The static rule** is that a ZWJ sequence measures as the width of its first
  segment — 2 — which is what a composing terminal does.
- **The calibration** (`caps_calibrate`, CAPABILITIES.md §5) writes a probe
  string to the terminal at startup, asks where the cursor ended up with
  `CSI 6 n` (DSR), and records the terminal's *actual* behaviour for four
  probes: a ZWJ family, a flag (regional indicator pair), a base + VS16, and a
  wide CJK character. The result is four booleans in `Caps`, and the width
  function consults them.
- **The escape hatch** is `Caps.assume_narrow_zwj`, settable by an application
  that knows better than either.

Calibration costs one round trip at startup, is bounded by
`NTUI_PROBE_TIMEOUT`, and is skipped entirely on the headless device (where the
answer is whatever the test says). An application may disable it; the default
is on, because a wrong width is a visible defect and a 100 ms startup cost is
not.

**Rule X-13 — a cluster wider than the space remaining is not truncated
mid-cluster.** A width-2 cluster with one column left is replaced by a single
space, and the fact is recorded so a debug build can report it. Writing half of
a wide character is what produces the classic shear.

---

## 5. The tables

**Rule X-14 (T-021).** The tables are **generated** from the Unicode Character
Database by `tools/gen_unicode.py` and **committed as Nitpick source** under
`src/unicode/`. A build needs the compiler and nothing else — no Python, no
network, no `/usr/share/unicode`.

**Rule X-15.** The generator is checked, not trusted: the harness re-runs it
and requires the committed tables to be **byte-identical** to what it would
emit. A generated file that has been hand-edited is the failure this prevents,
and it is the same instrument the compiler uses for its builtin signature
table.

**Rule X-16 — the pinned version.** The Unicode version is recorded in exactly
one place, `src/unicode/version.npk`, as a `pub fixed string:UNICODE_VERSION`,
and it appears in the header comment of every generated file. Upgrading it is a
recorded decision with a regenerated table set and a re-run of the golden
suite, because a width that changes is a rendering that changes.

> The version to pin is **the latest published UCD at the time cycle 0.2
> runs**, and the plan records the one actually used rather than guessing now.
> The floor is 15.1.0, below which `InCB` does not exist and rule GB9c cannot
> be implemented.

**Rule X-17 — the representation is a sorted range array with binary search.**

```nitpick
pub struct:Range32 = { uint32:lo; uint32:hi; };      // inclusive, sorted, disjoint
```

Four tables: `WIDE` (rules 5 and 6 merged), `ZERO` (rule 7),
`AMBIGUOUS` (rule 11), and `EMOJI` (for rule 3's base test). Two more for
segmentation: `GCB` (range → property byte) and `EXTPICT`.

Not a two-stage trie, and the reason is deliberate: a binary search over a
sorted disjoint range array has one invariant (`lo <= hi`, and each range's
`lo` greater than the previous `hi`), that invariant is checkable by a test
over the committed table in one pass, and the search's bound is a single
proof obligation for cycle 1.5. A trie is faster and has four invariants
nobody will state. If measurement later shows the search on the hot path, the
answer is a small direct-indexed fast path for `U+0000…U+02FF` — where the
overwhelming majority of characters live — in front of the same tables, and
not a change of representation.

**Rule X-18.** Every table function is `never fails` and total. `width(cp)`
answers for every `uint32`, including values above U+10FFFF (width 1, and the
decoder cannot produce them anyway).

---

## 6. What is deliberately absent

- **Normalisation** (NFC/NFD/NFKC/NFKD). A terminal displays what it is given;
  normalising a user's text behind their back changes their data. If a widget
  needs comparison-insensitive-to-normalisation it does so explicitly, and
  nothing in the library does today.
- **Case mapping** beyond ASCII. `to_upper` on Turkish `i` is a locale
  question, and this library has no locale.
- **Collation.** Sorting strings is the application's, and UCA is a large
  correct answer to a question a TUI toolkit is not asked.
- **Bidirectional text** (UAX #9). Stated as absent rather than forgotten: RTL
  in a cell grid is a genuinely hard problem — the terminal, not the
  application, owns the reordering on most emulators, and the two doing it at
  once is worse than either. `ntui` renders logical order and lets the terminal
  do what it does. Revisiting this is its own cycle and its own decision.

---

## 7. Open items

- **O-X1 — the Unicode version to pin.** Decided at cycle 0.2 against what is
  published then; floor 15.1.0. Tracked in `../OPEN_QUESTIONS.md`.
- **O-X2 — whether `width()` should take the cluster or the base codepoint in
  its public signature.** Cluster is correct (rules 3 and 4 need it) and costs
  a slice; codepoint is what callers reach for. Recommendation: the public API
  is `text_width(uint8[]) → uint32` over a whole string and
  `cluster_width(uint8[]) → uint8` over one cluster, and the codepoint form is
  internal — a caller who has a bare codepoint has already lost the
  information the rules need.
