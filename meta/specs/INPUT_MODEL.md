# Input

The event vocabulary, and the decoder that produces it. This is the largest
single piece of the library and the one with the most ways to be subtly wrong,
so it is specified byte by byte.

---

## 1. The events

```nitpick
pub enum:Event = {
    Key(KeyEvent);
    Mouse(MouseEvent);
    Paste(PasteEvent);        // §8 — a whole paste, never synthetic keys
    Resize(uint16, uint16);   // rows, cols
    FocusGained;
    FocusLost;
    Reply(ReplyEvent);        // §10 — what the terminal answered
    Signal(SignalEvent);      // §11 — from the signal fd, not from bytes
    Tick;                     // §12 — the runtime's frame/timer pulse
};
```

**Rule I-1.** `Event` is a plain value with **no owning fields**. It lives in a
fixed-capacity ring inside the decoder and is copied out by value, which is
what lets the ring be an array and the decoder be allocation-free. A paste's
bytes are the one thing too large to inline; §8 says where they live.

### 1.1 Keys

```nitpick
pub struct:KeyEvent = {
    KeyCode:code;
    uint8:mods;             // §3
    KeyKind:kind;           // Press | Repeat | Release
    char32:shifted;         // the shifted layout key, 0 when unknown (kitty)
    char32:base;            // the base layout key,    0 when unknown (kitty)
    uint8[8]:text;          // the text this press produced, UTF-8
    uint8:text_len;         // 0 when the press produced no text
};

pub enum:KeyKind = { Press; Repeat; Release; };
```

**Rule I-2.** `kind` is `Press` unless the kitty protocol reported otherwise.
A terminal that cannot report releases reports none, and an application that
requires them checks `Caps.kitty_keyboard` rather than waiting forever.

**Rule I-3 — `text` is the authority for "what did the user type".** A text
input widget appends `text`, and never reconstructs a character from `code` and
`mods`. On the kitty protocol `text` is what the terminal said; on the legacy
path it is the decoded UTF-8 of a printable key press with no Ctrl and no Alt.
`Ctrl+A` has `code = Char('a')`, `mods = CTRL`, and `text_len = 0` — because
Ctrl+A produced no text, and a widget that inserted `\x01` because it read the
byte is the bug this field exists to prevent.

`KeyCode` is a tagged enum. The full list is in §4; the shape is:

```nitpick
pub enum:KeyCode = {
    Char(char32);            // a printable key, by its unshifted codepoint
    F(uint8);                // F1 … F35
    Unknown(uint32);         // an unmapped kitty functional key, kept not dropped
    Enter; Tab; Backspace; Escape; Space;
    Left; Right; Up; Down; Home; End; PageUp; PageDown; Insert; Delete;
    // … §4
};
```

### 1.2 Mouse

```nitpick
pub struct:MouseEvent = {
    MouseKind:kind;
    MouseButton:button;
    uint16:col;             // 0-based CELL column
    uint16:row;             // 0-based CELL row
    uint16:xpixel;          // 0 unless SGR-pixel reporting is on
    uint16:ypixel;
    uint8:mods;
};
pub enum:MouseKind   = { Down; Up; Drag; Move; ScrollUp; ScrollDown; ScrollLeft; ScrollRight; };
pub enum:MouseButton = { NoButton; Left; Middle; Right; Button4; Button5; Button6; Button7; };
```

**Rule I-4 — coordinates are 0-based cells.** The wire is 1-based; the
conversion happens once, in the decoder, and every layer above works in the
same coordinate space the layout engine and the cell buffer use. A library that
leaks 1-based coordinates from the wire produces exactly one off-by-one per
application.

**Rule I-5.** A wheel event is `ScrollUp`/`ScrollDown`/`ScrollLeft`/
`ScrollRight` with `button = NoButton`, not a `Down` on `Button4`. The wire
encodes wheels as buttons 64–67; that is a wire fact and it stops at the
decoder.

---

## 2. The decoder is a pure function

**Rule I-6 (T-030).** The decoder performs no I/O, holds no descriptor, and
knows nothing about a terminal:

```nitpick
pub func:input_feed = NIL(Decoder->:d, uint8[]:bytes, int64:now_ns);
pub func:input_next = Event?(Decoder->:d, int64:now_ns);
```

`input_feed` consumes bytes and appends whatever complete events they produced
to the decoder's ring. `input_next` pops one, or `NIL` when the ring is empty.
`now_ns` is a monotonic timestamp the **caller** supplies, so that the escape
timeout (§6) is a function of a value a test can control rather than of the
clock.

Everything follows from this. The decoder is exercised entirely from committed
byte fixtures with no terminal, timing included; a captured session from a real
terminal replays deterministically; and a fuzzer can drive it directly.

**Rule I-7.** The decoder never fails on garbage. Unrecognised sequences are
consumed and dropped, and a counter of them is exposed for diagnostics. The one
failure it can report is `ETuiCapacity`: an event ring that overflowed, or a
sequence longer than the maximum (§13). A terminal that emits nonsense must not
be able to stop the program.

**Rule I-8 — no byte is consumed twice and none is lost.** The decoder holds a
partial-sequence buffer across `input_feed` calls, because a `read` may split a
sequence anywhere — a fact every implementation knows and half get wrong at
exactly one boundary. The fixture suite includes every sequence delivered
**one byte per feed**, which is the test that finds this.

### 2.1 The states

```
Ground ──0x1B──► Escape ──'['──► CsiParam ──►…──► (dispatch)
   │                │  ├─'O'──► Ss3
   │                │  ├─'P'──► DcsIgnore …ST
   │                │  ├─']'──► OscString …BEL|ST
   │                │  ├─'_' '^' 'X' ──► StrIgnore …ST
   │                │  └─ else ──► (Alt+key, or Escape by timeout)
   ├─ 0x00..0x1F ──► control key
   ├─ 0x20..0x7E ──► printable ASCII
   └─ 0x80..0xFF ──► Utf8Cont (1..3 more bytes)
```

Plus the sub-states `CsiParam` needs: an intermediate collector, a private-marker
flag (`?`, `<`, `>`, `=`), and the parameter list with sub-parameters
(`:`-separated, which the kitty protocol requires).

**Rule I-9.** Parameters are `uint32`, capped at 65535 on overflow rather than
wrapping, with at most 32 parameters and at most 8 sub-parameters each. A
sequence exceeding either is consumed and dropped. This is the ECMA-48 shape
with limits stated, and the limits are what make the parser's storage a fixed
array.

**Rule I-10 — a C0 byte inside a sequence is dispatched immediately and does
not abort the sequence**, which is ECMA-48's rule and what real terminals do.
`ESC [ 1 <0x07> ; 2 A` yields a bell (ignored) and then the modified Up arrow.
`0x18` (CAN) and `0x1A` (SUB) abort to Ground. A second `0x1B` restarts the
sequence.

---

## 3. Modifiers

**Rule I-11.** Modifiers are a `uint8` bitmask, and it is a plain integer with
named constants — **a library cannot declare a flag family** (D-230 makes
`TY_FLAGS` four compiler-known families whose members are generated prelude
constants), so `Mods` is not one and does not pretend to be.

| Constant | Bit | Wire (xterm `1 + n`) |
|---|---|---|
| `MOD_SHIFT` | 1 | 1 |
| `MOD_ALT` | 2 | 2 |
| `MOD_CTRL` | 4 | 4 |
| `MOD_SUPER` | 8 | 8 |
| `MOD_HYPER` | 16 | 16 |
| `MOD_META` | 32 | 32 |
| `MOD_CAPS_LOCK` | 64 | 64 |
| `MOD_NUM_LOCK` | 128 | 128 |

The wire encodes the set as `1 + bitmask` in a CSI parameter, so `CSI 1 ; 5 A`
is Ctrl+Up. `ntui` subtracts the one at the decoder and never again.

**Rule I-12.** `MOD_CAPS_LOCK` and `MOD_NUM_LOCK` are **state**, not modifiers,
and are reported only by the kitty protocol. They are excluded from the
comparison helper `mods_eq_ignoring_locks(a, b)`, which is what a key binding
should use, and included in the raw field, which is what a diagnostic should
print.

---

## 4. Key codes

The full `KeyCode` list, and where each comes from.

**Textual and basic** — every protocol: `Char(char32)`, `Enter`, `Tab`,
`Backspace`, `Escape`, `Space`.

**Navigation** — CSI and SS3: `Left`, `Right`, `Up`, `Down`, `Home`, `End`,
`PageUp`, `PageDown`, `Insert`, `Delete`, `Begin` (the keypad 5 / `CSI E`).

**Function** — `F(1)` … `F(35)`.

**Keypad, distinguished from the main block** — SS3 in application mode, and
the kitty functional codes: `KpEnter`, `Kp0`…`Kp9`, `KpDecimal`, `KpDivide`,
`KpMultiply`, `KpSubtract`, `KpAdd`, `KpEqual`, `KpSeparator`, `KpLeft`,
`KpRight`, `KpUp`, `KpDown`, `KpHome`, `KpEnd`, `KpPageUp`, `KpPageDown`,
`KpBegin`, `KpInsert`, `KpDelete`.

**Modifier keys as keys** — kitty only: `LeftShift`, `LeftCtrl`, `LeftAlt`,
`LeftSuper`, `LeftHyper`, `LeftMeta`, and the six `Right…` twins; `CapsLock`,
`NumLock`, `ScrollLock`.

**System** — kitty and some CSI: `PrintScreen`, `Pause`, `Menu`.

**Media** — kitty only: `MediaPlay`, `MediaPause`, `MediaPlayPause`,
`MediaReverse`, `MediaStop`, `MediaFastForward`, `MediaRewind`,
`MediaTrackNext`, `MediaTrackPrevious`, `MediaRecord`, `VolumeDown`,
`VolumeUp`, `VolumeMute`.

**Unmapped** — `Unknown(uint32)` carries the kitty functional codepoint we did
not recognise. It is kept rather than dropped so that a terminal ahead of our
table is diagnosable rather than silent.

### 4.1 The control-byte table (Ground state, `0x00`…`0x1F`)

Where two spellings collide the table says which wins, and why.

| Byte | `KeyCode` | `mods` | Note |
|---|---|---|---|
| `0x00` | `Char(' ')` | `CTRL` | Ctrl+Space; Ctrl+@ is the same byte and the space spelling is the one users mean |
| `0x01`…`0x07` | `Char('a'…'g')` | `CTRL` | |
| `0x08` | `Backspace` | `CTRL` | Ctrl+H. See I-13. |
| `0x09` | `Tab` | — | Ctrl+I is the same byte; `Tab` wins |
| `0x0A` | `Char('j')` | `CTRL` | Ctrl+J. **Not** Enter — see I-13 |
| `0x0B`…`0x0C` | `Char('k'…'l')` | `CTRL` | |
| `0x0D` | `Enter` | — | Ctrl+M is the same byte; `Enter` wins |
| `0x0E`…`0x1A` | `Char('n'…'z')` | `CTRL` | |
| `0x1B` | — | — | escape introducer; §6 |
| `0x1C` | `Char('\\')` | `CTRL` | |
| `0x1D` | `Char(']')` | `CTRL` | |
| `0x1E` | `Char('^')` | `CTRL` | |
| `0x1F` | `Char('/')` | `CTRL` | Ctrl+/ and Ctrl+_ are one byte; `/` is what users press |
| `0x7F` | `Backspace` | — | See I-13 |

**Rule I-13 — the Backspace/Enter question, settled.** `0x7F` (DEL) is
`Backspace` and `0x08` (BS) is `Backspace` with `CTRL`. This is the modern
convention, it matches what terminals send for the Backspace key
(`0x7F` on essentially all of them, because `erase = ^?`), and it makes Ctrl+H
distinguishable — which the alternative does not. Similarly `0x0D` is `Enter`
and `0x0A` is Ctrl+J: with `ICRNL` cleared (TERMINAL_MODEL.md §3) the Enter
key sends `0x0D` and nothing else sends it, so the mapping is exact rather
than a guess.

---

## 5. Sequences

### 5.1 SS3 — `ESC O <final>`

Sent by terminals in application-cursor-key or application-keypad mode.

| Final | Key | Final | Key |
|---|---|---|---|
| `A` `B` `C` `D` | Up Down Right Left | `P` `Q` `R` `S` | F1 F2 F3 F4 |
| `H` `F` | Home End | `M` | `KpEnter` |
| `E` | `Begin` | `X` | `KpEqual` |
| `j`…`y` | keypad `*` `+` `,` `-` `.` `/` `0`…`9` | | |

`ESC O <mods> <final>` (some terminals) applies the §3 mask.

### 5.2 CSI, cursor and edit finals

`CSI [<params>] <final>` where a missing parameter defaults to 1.

| Final | Key | Modified form |
|---|---|---|
| `A` `B` `C` `D` | Up Down Right Left | `CSI 1 ; m A` |
| `H` `F` | Home End | `CSI 1 ; m H` |
| `E` | `Begin` | |
| `P` `Q` `S` | F1 F2 F4 | some terminals; `R` is **not** used — it is the CPR reply |
| `Z` | `Tab` + `SHIFT` (back-tab) | |
| `u` | kitty (§5.5) **or** `modifyOtherKeys` (§5.4) — decided by parameter count | |

**Rule I-14 — `CSI R` is a cursor-position report, never F3.** Some historical
terminals sent `CSI R` for F3; every terminal in existence sends `CSI r ; c R`
for DSR, and `ntui` needs DSR for width calibration. `CSI R` with two
parameters is a `Reply`; with none it is dropped. F3 arrives as `CSI 1 3 ~` or
`ESC O R` on the terminals that send it.

### 5.3 CSI tilde keys — `CSI <n> [; <m>] ~`

| n | Key | n | Key | n | Key |
|---|---|---|---|---|---|
| 1 | Home | 15 | F5 | 24 | F12 |
| 2 | Insert | 17 | F6 | 25 | F13 |
| 3 | Delete | 18 | F7 | 26 | F14 |
| 4 | End | 19 | F8 | 28 | F15 |
| 5 | PageUp | 20 | F9 | 29 | F16 |
| 6 | PageDown | 21 | F10 | 31 | F17 |
| 7 | Home | 23 | F11 | 32 | F18 |
| 8 | End | | | 33 | F19 |
| 11…14 | F1…F4 | | | 34 | F20 |

`200` and `201` are bracketed paste and are not keys (§8).

### 5.4 `modifyOtherKeys` — `CSI 27 ; <m> ; <c> ~`

xterm's answer to "Ctrl+Shift+A is indistinguishable from Ctrl+A". `c` is the
Unicode codepoint of the unshifted key, `m` the §3 mask. `ntui` requests level
2 when `Caps.modify_other_keys` and the kitty protocol is unavailable, and
decodes it always.

### 5.5 The kitty keyboard protocol — `CSI <key> [: <alt>] [; <mods> [: <event>]] [; <text>] u`

The preferred encoding (T-032), because it is the only one that reports
releases, distinguishes every modifier combination, and carries the text the
key produced.

- **`key`** is the unshifted Unicode codepoint, or a functional-key code from
  the protocol's private range (`57344`…`63743`).
- **`alt`** is `shifted_key : base_layout_key`, either part optionally empty.
- **`mods`** is `1 + mask` per §3.
- **`event`** is 1 press, 2 repeat, 3 release.
- **`text`** is one or more codepoints, `:`-separated, that the press produced.

**Rule I-15 — the flags `ntui` pushes are `1 | 2 | 8 | 16` = 27**
(`0b11011`): bit 1 *disambiguate escape codes*, bit 2 *report event types*,
bit 8 *report all keys as escape codes*, bit 16 *report associated text*.

Bit 4 (*report alternate keys*) is **not** pushed by default: it adds the
shifted and base-layout codepoints to every event, only a keyboard-remapping
application needs them, and `Caps.kitty_flags` can add it.

**Bit 16 is requested because bit 8 is, and the pairing is not optional.** With
bit 8 set the terminal stops sending plain text bytes and the `text` field
becomes the only source of what the user typed, so requesting one without the
other produces an application that cannot type. This is the single most common
way to get this protocol wrong, and it is why the two are one constant here
rather than two settings.

**Rule I-16 — the decoder accepts every kitty shape regardless of what was
pushed.** A terminal may be in a mode a previous program left, an outer
multiplexer may add flags, and a fixture may exercise a shape we do not
request. The decoder's job is to read what arrives.

### 5.6 Mouse

**SGR (1006)** — the only encoding `ntui` requests (T-033):

```
CSI < <b> ; <col> ; <row> M     press or motion
CSI < <b> ; <col> ; <row> m     release
```

`b` decomposes as: low two bits the button (0 left, 1 middle, 2 right, 3 none),
`+4` Shift, `+8` Alt (Meta), `+16` Ctrl, `+32` motion, `+64` wheel (then the
low two bits select up/down/left/right), `+128` buttons 8–11.

**SGR-pixel (1016)** is the same with pixel coordinates; `ntui` requests it
only when `Caps.mouse_sgr_pixel`, and then fills both the cell fields (derived
by dividing by the cell size from `TIOCGWINSZ`'s pixel fields when available)
and the pixel fields.

**X10 (`CSI M <b+32> <x+32> <y+32>`)** and **UTF-8 (1005)** and
**urxvt (1015)** are **decoded** — a terminal may be left in one of these modes
by a previous program — and never **requested**. X10 is the one encoding whose
coordinates break above 223, which is exactly why it is not requested.

### 5.7 Focus

`CSI I` is `FocusGained`, `CSI O` is `FocusLost`. Requested only when
`Caps.focus_events`.

---

## 6. The escape ambiguity

A lone `ESC` is the Escape key. `ESC` followed by `[` begins a sequence. `ESC`
followed by a printable byte is Alt+that key. The bytes are identical and the
terminal offers no framing.

**Rule I-17 (T-031) — resolved by an explicit deadline, never by a heuristic.**
On `ESC` with no following byte in the same feed, the decoder records the
timestamp it was given. If `input_next` is called at a `now_ns` more than
`NTUI_ESC_TIMEOUT` (25 ms, `SAFETY.md` §4) later and no continuation has
arrived, the `ESC` is emitted as `Escape` and the state returns to Ground.

Three consequences, each stated because each is a real complaint about some
other library:

- **The timeout is a parameter, not a constant in the code.** Over a slow SSH
  link 25 ms is too short and the Escape key becomes an Alt-key soup; the
  application raises it.
- **A continuation that arrives late produces the sequence anyway**, not a
  spurious `Escape` followed by garbage — because the timeout only fires when
  the caller asks and nothing has arrived, and a caller that is polling fast is
  not accumulating.
- **The kitty protocol removes the ambiguity entirely.** With bit 1
  (disambiguate) pushed, Escape arrives as `CSI 27 u` and the timeout is never
  consulted. When `Caps.kitty_keyboard` is on, `NTUI_ESC_TIMEOUT` is set to
  zero and a lone `ESC` resolves immediately.

**Rule I-18.** `ESC ESC` is Escape then Escape, not Alt+Escape. Two presses is
what the user did.

---

## 7. Alt

**Rule I-19.** `ESC` + a printable byte or a UTF-8 sequence is that key with
`MOD_ALT`. `ESC` + a control byte is that control key with `MOD_ALT` — so
`ESC 0x01` is Ctrl+Alt+A. This is the "Alt sends Escape" convention, which is
every terminal's default; the alternative (setting the eighth bit) is decoded
too, when a byte in `0x80`…`0xFF` is not a valid UTF-8 lead.

---

## 8. Paste

**Rule I-20 (T-035).** Between `CSI 200 ~` and `CSI 201 ~`, bytes are paste
content and are **not** interpreted. No key events are produced, no escape
sequence inside is acted on, and the terminator is matched literally.

**Rule I-21 — a paste is one event.** The content accumulates in the decoder's
paste buffer — a `Bytes` owned by the `Decoder` — and the `Paste` event carries
the offset and length of that content within it, valid until the next
`input_next`. This is the one place `Event` refers to storage rather than
containing it, and it is done that way because pastes are routinely megabytes
and an event ring of fixed-size values cannot hold one.

**Rule I-22 — the paste buffer is bounded.** Default 1 MiB, configurable. A
paste exceeding it is **truncated**, the `Paste` event carries `truncated =
true`, and the remaining bytes up to the terminator are consumed and dropped.
It is not an error: a user pasting a large file into a text field should get a
usable prefix and a signal, not a program that stops.

**Rule I-23.** The content is **not** validated as UTF-8 by the decoder. A
paste can be anything. The widget that inserts it validates or substitutes,
per `TEXT_MODEL.md` §4.

**Rule I-24 — an unterminated paste ends at end of input**, producing the
`Paste` event with what was gathered, so a terminal that dies mid-paste does
not leave the decoder wedged in a state that swallows every subsequent
keystroke.

---

## 9. Signals as events

**Rule I-25.** `SIGWINCH` becomes `Resize` after a fresh `TIOCGWINSZ`
(TERMINAL_MODEL.md §6), not directly. `SIGTERM`, `SIGHUP`, `SIGINT` and
`SIGQUIT` become `Signal(…)`, which an application handles or ignores — but
`SIGHUP` additionally sets a device flag that makes every subsequent write a
no-op, because the terminal is gone and writing to it is pointless.
`SIGTSTP`/`SIGCONT` are handled by the runtime (TERMINAL_MODEL.md §8) and
surface as `Signal(Suspended)` / `Signal(Resumed)` after the fact.

---

## 10. Replies

**Rule I-26 (T-034).** Everything the terminal answers is a `Reply` event, and
`Reply` events reach the application unless something asked for them.

```nitpick
pub enum:ReplyEvent = {
    DeviceAttributes(uint32, uint32);   // DA1: the first two parameters
    CursorPosition(uint16, uint16);     // DSR/CPR, converted to 0-based
    ModeReport(uint32, uint8);          // DECRQM: mode, state
    KittyFlags(uint8);
    TextArea(uint16, uint16);           // CSI 8 ; r ; c t
    Version(uint8[32], uint8);          // XTVERSION, truncated, with its length
    Osc(uint16, uint8[64], uint8);      // OSC number, truncated payload, length
    Unrecognised(uint8);                // the final byte, for diagnostics
};
```

**Rule I-27 — the probe consumes what it asked for and nothing else.** The
capability probe (`CAPABILITIES.md` §4) reads events until DA1 and takes the
replies it recognises; every other event, key presses included, is pushed back
onto the front of the ring in order. A terminal library that discards the
keystrokes a user typed during startup is a library people notice.

**Rule I-28 — DCS, OSC, APC, PM and SOS strings are length-bounded and
terminated by `ST` (`ESC \`) or, for OSC only, `BEL`.** Content beyond the
bound is dropped and the sequence still terminates correctly, so an
unterminated string cannot consume the session. The bound is 4096 bytes.

---

## 11. What is deliberately not decoded

- **DECRPSS / DECRQSS replies** beyond the truecolour probe's one use
  (`CAPABILITIES.md` §6): they arrive as `Reply.Unrecognised`.
- **OSC 52 clipboard content**: the *reply* is decoded to `Reply.Osc` and left
  there. A clipboard API is a separate feature with its own security questions
  and its own decision.
- **Sixel and Kitty graphics responses**: recorded as `Reply.Unrecognised`
  until an image cycle exists.

---

## 12. `Tick`

Not from bytes. The runtime emits `Tick` when a frame is due or an application
timer expires (`EVENT_MODEL.md` §4). It is in this enum rather than a parallel
channel because an application's `update` should see one ordered event stream —
that ordering is what makes a replay test possible.

---

## 13. Limits

| Thing | Bound | On exceeding |
|---|---|---|
| event ring | 256 events | `ETuiCapacity`; the runtime drains before feeding, so this means the application stalled |
| CSI parameters | 32 | sequence dropped |
| sub-parameters per parameter | 8 | sequence dropped |
| parameter value | 65535 | clamped |
| string sequence (DCS/OSC/APC/PM/SOS) | 4096 bytes | content truncated, sequence still terminated |
| pending partial sequence | 128 bytes | reset to Ground, counted |
| paste buffer | 1 MiB, configurable | truncated, flagged (I-22) |

**Rule I-29.** Every bound is a named constant in one file, every one is
exercised by a fixture that sits exactly on it and one that exceeds it, and
none of them is a `while` loop over attacker-controlled length without one.

---

## 14. Testing

Summarised here; [`TESTING.md`](TESTING.md) §4 has the mechanism.

1. **A table-driven suite**: every sequence in §§4–8, its expected event, and
   the same bytes fed **one at a time**.
2. **Captured fixtures**: real byte streams from every terminal in
   `COMPAT.md`'s matrix, recorded by a small capture tool
   (`tools/capture.npk`), committed under `tests/fixtures/input/`, with the key
   or gesture that produced them named in the file. This is the corpus that
   makes a compatibility claim checkable rather than asserted.
3. **A round-trip property**: for every event the library can *emit*
   (the golden oracle's synthetic input), decoding what a terminal would send
   for it yields the same event.
4. **A fuzzer**: random bytes, and structured mutations of the fixtures, with
   the invariants "never traps", "never consumes a byte twice", "never
   allocates unboundedly", and "always returns to Ground within the pending
   bound".

---

## 15. Open items

- ~~**O-I1 — shifted or unshifted `Char` on the legacy path.**~~ — **SETTLED
  (T-108).** The legacy path reports `Char('A')` with **no** `MOD_SHIFT`:
  inventing a modifier the terminal did not report is a lie a key binding will
  trip over. The asymmetry between protocols is documented, and
  `key_matches()` absorbs it **at the point of use** — a binding asks "is this
  Shift+A" and the helper answers correctly under either encoding.
- ~~**O-I2 — whether to request kitty bit 4** (report alternate keys) by
  default.~~ — **SETTLED (T-109): no.** The cost is on every keystroke and only
  a keyboard-remapping application needs them; §5.5 has the reasoning and
  `Caps.kitty_flags` is the opt-in.
- **O-I3 — key repeat on the legacy path.** Indistinguishable from a fast
  press; reported as `Press`. Only the kitty protocol can do better and it
  does.
