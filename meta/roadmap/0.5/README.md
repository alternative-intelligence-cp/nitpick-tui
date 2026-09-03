# Cycle 0.5 — Style

**`src/style/`: `Color`, `Style`, `StylePatch`, attributes, the degradation
tables, and the SGR emitter.** Pure value work with one output: bytes.

## Decisions in

T-040, T-041, T-042. Settled. **No open questions.**

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| 0.5.0 | **The values** — `Color`, `ColorKind`, `Style`, the attribute constants, the underline-style encoding | 16-byte `Style` with `#size_of` asserted, no owning field |
| 0.5.1 | **Degradation** — the RGB→256 mapping, the generated 256→16 and RGB→16 tables | the 2^24 walk green |
| 0.5.2 | **Emission** — the pen, T-042's transition rule, the colon/semicolon form, hyperlinks | every transition in a generated cross-product asserted byte for byte |
| 0.5.3 | **Patches** — `StylePatch`, `style_patch`, associativity | the algebra's laws proven over a generated corpus |
| 0.5.4 | **Close** | `done/0.5/`, `0.6.0.md` written |

## Checklist

### 0.5.0 — the values
- [ ] `Color` is 4 bytes: `kind`, `a`, `b`, `c`; `Default`, `Ansi`, `Indexed`, `Rgb`
- [ ] `Ansi` and `Indexed` are distinct kinds and emit different SGR (T-040) — a test asserts the two byte streams differ
- [ ] the sixteen named constructors, plus `color_default`, `color_ansi`, `color_indexed`, `color_rgb`
- [ ] `Style` is exactly 16 bytes; `#size_of<Style>()` asserted
- [ ] `attrs` is `uint16` with bits 0…9 boolean and 12…14 the underline style
- [ ] every attribute constant and its on/off SGR from `specs/STYLE_MODEL.md` §3
- [ ] `link` is a `uint16` id, 0 meaning none
- [ ] `check_no_owning_cells` covers `Style`

### 0.5.1 — degradation
- [ ] the xterm cube levels `0, 95, 135, 175, 215, 255` and the grey ramp `8 + 10i`
- [ ] both candidates computed, smaller squared distance wins, ties break **toward the cube** (Y-7)
- [ ] the 256→16 table generated into `src/style/degrade.npk` and regeneration-checked
- [ ] RGB→16 as its own table, so it does not lose twice
- [ ] **the 2^24 walk** (Y-9): every RGB value, asserting the result is in range, that exact cube colours map to themselves, and that exact greys take the ramp
- [ ] `None` drops colour and keeps attributes
- [ ] an unsupported attribute is **dropped, never substituted** (Y-8); a curly underline becomes a plain one, a plain one becomes nothing, and neither becomes reverse video

### 0.5.2 — emission
- [ ] the pen tracked; only transitions emitted (Y-10)
- [ ] **T-042's rule**: additions only when the target's attribute set is a superset and neither colour becomes `Default`; otherwise `CSI 0 m` and the target in full
- [ ] a test over the generated cross-product of (pen, target) pairs, each asserted byte for byte
- [ ] a test that specifically covers the bold/dim case Y-5 names: bold+dim → dim alone must reset and reapply
- [ ] one SGR per run, codes joined
- [ ] colon and semicolon colour forms, chosen by capability, emitted by one function from one table (Y-12)
- [ ] the pen reset at frame start (Y-13)
- [ ] OSC 8 open/close on link-id change, dropped without the capability, no `id=` parameter (Y-14)

### 0.5.3 — patches
- [ ] `StylePatch` with `add`/`remove`/`set`, and **no sentinel for "unspecified colour"** — the second type is the whole point (Y-15)
- [ ] `style_patch(base, over)` field by field
- [ ] associativity and identity proven over a generated corpus (Y-16)

## Gate

The 2^24 degradation walk, and the transition cross-product asserted byte for
byte.

## Watch for

- **`SGR 22` turns off bold and dim together.** That single fact is why T-042
  is what it is, and the test that covers it is the one that would be forgotten.
- **A `Style` always means what its author wrote.** Degradation happens in the
  renderer at emission (C-14), never in the value, so the same `Style` on two
  terminals is the same value with two byte streams — which is what a test can
  see.
