# Cycle 0.3 — Input

**`src/input/`: the decoder — a pure incremental function from bytes to
events.** The largest single piece of the library and the one with the most
ways to be subtly wrong.

## Why here

Because capabilities need it (a terminal's replies are events, T-034) and the
runtime needs both. And because it is entirely testable with no terminal: the
decoder holds no descriptor and takes its clock from its caller (T-030), so
this cycle's whole suite is fixtures and property tests.

## Decisions in

T-030 … T-037, T-041, T-090. Settled.

**Open questions to settle:** O-I1 (shifted vs unshifted `Char` on the legacy
path), O-I2 (kitty bit 4 by default). Recommendations on file.

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| 0.3.0 | **The vocabulary** — `Event`, `KeyEvent`, `KeyCode`, `MouseEvent`, `ReplyEvent`, `Mods`, and the event ring | the types, POD, in a fixed array, with `#size_of` asserted |
| 0.3.1 | **The state machine** — Ground, Escape, CSI, SS3, DCS/OSC/APC/PM/SOS, UTF-8 continuation; the parameter parser with sub-parameters | every state exercised, every bound in §13 sat on and exceeded |
| 0.3.2 | **Keys** — the control-byte table, the CSI and SS3 tables, tilde keys, `modifyOtherKeys`, Alt, the escape timeout | every sequence in `specs/INPUT_MODEL.md` §§4–7 |
| 0.3.3 | **Kitty** — the full `CSI … u` form: alternates, event types, associated text | the flags pairing (I-15) proven by a test that shows the wrong pairing breaks typing |
| 0.3.4 | **Mouse** — SGR, SGR-pixel, and the decoded-but-never-requested legacy encodings | every button, every modifier, every wheel direction, drag and motion |
| 0.3.5 | **Paste, focus, replies** — bracketed paste with its bound and truncation, focus in/out, the whole `ReplyEvent` set | a 2 MiB paste truncated and flagged, and an unterminated paste that does not wedge |
| 0.3.6 | **The corpus** — `tools/capture.npk`, the first fixture directories, the byte-at-a-time re-run | fixtures for at least four real terminals |
| 0.3.7 | **The fuzzer** — `tools/fuzz_input.py`, the four invariants, the committed corpus | a million inputs with no trap |
| 0.3.8 | **Close** | `done/0.3/`, `0.4.0.md` written |

## Checklist

### 0.3.0 — the vocabulary
- [ ] `Event` and every payload type declare **no owning field** — `check_no_owning_cells` goes live here
- [ ] the event ring is a fixed 256-entry array, and overflow is `ETuiCapacity` (I-7)
- [ ] `Mods` is a plain `uint8` with named constants (T-041), and `mods_eq_ignoring_locks` exists (I-12)
- [ ] `KeyEvent.text` is `uint8[8]` + a length, and `Ctrl+A` has `text_len == 0` (I-3)
- [ ] `KeyCode`'s full list from §4, with `Unknown(uint32)` for unmapped kitty codes
- [ ] `#[derive(Eq, Debug)]` on the event types, and `#size_of` asserted for each

### 0.3.1 — the state machine
- [ ] every state and transition in §2.1
- [ ] the parameter parser: 32 parameters, 8 sub-parameters, values clamped at 65535 (I-9)
- [ ] a C0 byte inside a sequence dispatches immediately without aborting it; `CAN` and `SUB` abort to Ground; a second `ESC` restarts (I-10)
- [ ] the partial-sequence buffer carried across `input_feed` calls (I-8), bounded at 128 bytes
- [ ] **every bound sat on exactly and exceeded by one**, with the documented behaviour each time
- [ ] the decoder never fails on garbage; unrecognised sequences are counted (I-7)

### 0.3.2 — keys
- [ ] the control-byte table, every row, including the three collisions T-036 settles
- [ ] SS3, every final in §5.1
- [ ] CSI cursor and edit finals, plain and modified
- [ ] the tilde table, every `n` in §5.3
- [ ] `CSI R` decoded as a cursor-position **reply**, never as F3 (I-14)
- [ ] `modifyOtherKeys` (`CSI 27 ; m ; c ~`)
- [ ] Alt: `ESC` + printable, `ESC` + UTF-8, `ESC` + control, and the eighth-bit form (I-19)
- [ ] `ESC ESC` is two Escapes (I-18)
- [ ] the escape timeout, driven by the caller's `now_ns`, at exactly the threshold and one nanosecond either side
- [ ] O-I1 decided and recorded

### 0.3.3 — kitty
- [ ] every field of `CSI key : alt ; mods : event ; text u`, in every combination of present and absent
- [ ] the functional-key range 57344…63743 mapped, with `Unknown` for the rest
- [ ] press, repeat and release
- [ ] **the pairing test** (I-15): a decoder fed a stream from a terminal with bit 8 and without bit 16 produces no text, demonstrating why both are pushed together
- [ ] the decoder accepts kitty shapes regardless of what was requested (I-16)
- [ ] O-I2 decided and recorded

### 0.3.4 — mouse
- [ ] SGR (1006): every button 0…2, buttons 8…11, the wheel's four directions, motion, drag, and the three modifier bits
- [ ] press and release distinguished by the `M`/`m` final
- [ ] coordinates converted to 0-based cells **at the decoder** (T-037)
- [ ] SGR-pixel (1016), with cell coordinates derived from the pixel size when known
- [ ] X10, 1005 and urxvt 1015 decoded; a test asserts none of them is ever *requested*
- [ ] a wheel is `ScrollUp`/`ScrollDown`, not a button press (I-5)

### 0.3.5 — paste, focus, replies
- [ ] bracketed paste: nothing inside is interpreted, the terminator matched literally
- [ ] the paste buffer bounded at 1 MiB, truncation flagged (I-22), configurable
- [ ] an unterminated paste ends at end of input and does not wedge the decoder (I-24)
- [ ] paste content **not** validated as UTF-8 by the decoder (I-23)
- [ ] `CSI I` / `CSI O` focus events
- [ ] every `ReplyEvent` variant: DA1, CPR, DECRQM, kitty flags, text area, XTVERSION, OSC
- [ ] DCS/OSC/APC/PM/SOS bounded at 4096 with correct termination on `ST` and, for OSC, `BEL` (I-28)

### 0.3.6 — the corpus
- [ ] `tools/capture.npk`: raw mode, read until a sentinel, write a named fixture
- [ ] fixture directories for at least four terminals from `specs/COMPAT.md`'s matrix
- [ ] every fixture decoded to the event its filename names
- [ ] **every fixture re-run one byte per `input_feed`**, same events — this is the test that finds the partial-sequence bugs
- [ ] `specs/COMPAT.md`'s matrix gains its first Verified rows, each with a fixture directory behind it (T-094)

### 0.3.7 — the fuzzer
- [ ] `tools/fuzz_input.py`: random bytes and structured mutations of the fixtures
- [ ] the four invariants: never traps; always returns to Ground within the pending bound; consumes every byte exactly once; allocation bounded by the paste buffer
- [ ] a million inputs clean
- [ ] any input that found something committed as a permanent fixture

## Gate

Every fixture decodes correctly **fed one byte at a time**, and the fuzzer runs
a million inputs without a trap.

## Watch for

- **`in` is a keyword.** A decoder wants a byte source called `in` constantly;
  it cost the compiler's 1.5.0 executor a build.
- **The partial-sequence case is the whole cycle.** A `read` splits a sequence
  wherever it likes and the byte-at-a-time re-run is the only test that finds
  it. Write it in 0.3.1, not at the end.
- **The escape timeout takes its clock from the caller** (T-030). A decoder
  that called `mono_now()` itself would be untestable at the boundary, which is
  the only place it matters.
- **Do not let the probe consume keystrokes.** I-27's push-back rule is
  implemented in 0.4, but the ring has to support it, so 0.3.0 designs for it.
