# Cycle 0.2 — Text

**`src/unicode/` and `src/core/utf8.npk`: decoding, grapheme segmentation, and
display width, from generated and committed tables.**

## Why here

Because every layer above it measures text, and because the failure it prevents
is the most visible one a terminal program has: a width wrong by one puts every
subsequent cell on the line in the wrong place, and a box drawn around it tears.
It is also almost entirely pure computation, so it is the cycle with the
cleanest tests in the plan.

## Decisions in

T-020 … T-025, T-093. Settled.

**Open questions to settle:** Q-1 / O-X1 (the Unicode version to pin) and O-X2
(cluster or codepoint in the public width signature).

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| 0.2.0 | **The generator** — `tools/gen_unicode.py`, the pinned UCD, the emitted table shape, the regeneration check | tables that regenerate byte-identically |
| 0.2.1 | **UTF-8** — the decoder DFA, the maximal-subpart replacement, the encoder | the decoder green on a full well-formedness corpus |
| 0.2.2 | **Width** — the four tables, the algorithm, the ambiguous capability | every rule in `specs/TEXT_MODEL.md` §4 asserted separately |
| 0.2.3 | **Segmentation** — UAX #29 extended grapheme clusters, incremental with carried state | `GraphemeBreakTest.txt`, every case |
| 0.2.4 | **Close** | `done/0.2/`, `0.3.0.md` written |

## Checklist

### 0.2.0 — the generator
- [ ] Q-1 answered: the UCD version pinned, recorded in `src/unicode/version.npk`
- [ ] `tools/gen_unicode.py` reads `UnicodeData.txt`, `EastAsianWidth.txt`, `emoji-data.txt`, `DerivedCoreProperties.txt`, `GraphemeBreakProperty.txt`
- [ ] the UCD files themselves are **gitignored** (they are large and reproducible); the generated tables are **committed**
- [ ] the emitted shape is `Range32 { uint32:lo; uint32:hi; }` arrays, sorted and disjoint (X-17)
- [ ] six tables: `WIDE`, `ZERO`, `AMBIGUOUS`, `EMOJI`, `GCB`, `EXTPICT`
- [ ] every generated file carries the Unicode version and the generator's name in its header
- [ ] `check_tables_regenerate` live and **seen to fail** against a hand-edited table
- [ ] `check_table_invariants` live: sorted, disjoint, `lo <= hi`, and every range within `U+10FFFF`

### 0.2.1 — UTF-8
- [ ] the decoder accepts exactly Unicode Table 3-7's well-formed sequences: no overlongs, no surrogates, nothing above U+10FFFF, per-lead-byte continuation ranges
- [ ] the maximal-subpart replacement (X-3): `E1 80 41` yields U+FFFD then `A`
- [ ] a corpus test over every 1-, 2-, 3- and 4-byte sequence's boundaries
- [ ] the encoder writes into a caller-supplied `uint8[4]` and returns the length, allocation-free
- [ ] round-trip: decode(encode(cp)) == cp for every scalar value
- [ ] `prove`-shaped comments on both: the consumed length is 1…4, the codepoint is a scalar value

### 0.2.2 — width
- [ ] the algorithm's eight steps implemented in order, each with its own test
- [ ] controls refused; `U+0000` handled; VS16 and VS15; `Emoji_Presentation`; EAW W/F; `Mn`/`Me`/`Cf`/`Default_Ignorable`; the default 1
- [ ] `Caps.ambiguous_wide` promotes the `A` class, and defaults off
- [ ] `text_width(uint8[])` and `cluster_width(uint8[])` — O-X2 decided
- [ ] the ZWJ static rule (2), with the calibration hooks present and unwired (0.4 wires them)
- [ ] `width()` is total: it answers for every `uint32`, including values above U+10FFFF
- [ ] a test that walks every codepoint 0…0x10FFFF and asserts the result is 0, 1 or 2 — nothing else, ever

### 0.2.3 — segmentation
- [ ] rules GB1–GB5, GB6–GB8 (Hangul), GB9, GB9a, GB9b, GB9c (Indic conjunct), GB11 (emoji ZWJ), GB12/GB13 (regional indicators), GB999
- [ ] the incremental signature `(state, bytes) → (cluster_end, state)` (X-7)
- [ ] the regional-indicator **parity** carried in the state (X-8) — the rule every hand-rolled segmenter gets wrong
- [ ] **the gate**: `GraphemeBreakTest.txt` from the pinned UCD, every case, no exceptions
- [ ] the same corpus fed **one byte at a time**, same answers
- [ ] a `prove`-shaped comment: the cluster end is strictly greater than the start, so the loop terminates (P-7)

### 0.2.4 — close
- [ ] O-X1 and O-X2 recorded as decisions
- [ ] findings written; `0.3.0.md` written; archived

## Gate

`GraphemeBreakTest.txt` green in full, and the width function total over the
whole codepoint space.

## Watch for

- **`trit` and `nit` are type keywords.** So are `unit` and `limit`. A Unicode
  module wants all four as names.
- **The tables are `fixed` module state**, so they are read-only memory and
  cost nothing at run time. A table built at startup would be a different and
  worse thing.
- **`GraphemeBreakTest.txt` is the only honest gate here.** A hand-written case
  set for UAX #29 tests the rules the author understood, which is the subset
  that was already right.
