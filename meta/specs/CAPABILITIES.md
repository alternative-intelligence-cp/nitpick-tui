# Capabilities

What the terminal can do, how `ntui` finds out, and why it does not read
terminfo.

---

## 1. The decision, and the argument for it

**T-003: `ntui` carries a compiled-in capability table and corrects it by
asking the terminal at startup. It never reads terminfo.**

The case against terminfo, in the order the reasons matter here:

1. **It is a binary format read from files the program does not control.** A
   parser for it is an untrusted-input parser inside a library whose entire
   claim is that it has no such surface. `ncurses` has had CVEs in exactly this
   code. There is nothing to gain that is worth adding it back.
2. **Its answers depend on machine state.** The same program on two machines
   with the same terminal emits different bytes, because `/usr/share/terminfo`
   differs, because `$TERM` was set to something approximate, or because the
   entry was compiled twenty years ago. `ntui`'s rendering is deterministic
   (`SCREEN_MODEL.md` §1) and terminfo would make it not.
3. **It is routinely wrong about modern terminals.** `xterm-256color` — what
   most terminals still claim to be — describes neither truecolor, nor
   synchronized output, nor the kitty keyboard protocol, nor SGR mouse, nor
   bracketed paste, nor styled underlines. Every terminal library that uses
   terminfo also carries a table of exceptions for exactly these, which means
   the table is doing the work and terminfo is doing the paperwork.
4. **The sequences that matter are standardised in practice.** The set `ntui`
   emits is ECMA-48 plus a small, well-documented body of DEC private modes and
   xterm extensions, and it is the same set on every terminal that supports
   them at all.

What is given up: exotic and historical terminals. A hardware VT52, a Wyse 60,
a line printer. `ntui` targets terminals that speak ECMA-48 with ANSI colour,
which is every terminal emulator in use, and says so in
[`COMPAT.md`](COMPAT.md) rather than pretending to a breadth it does not test.

---

## 2. The capability set

```nitpick
pub struct:Caps = {
    ColorDepth:color;          // None | Ansi16 | Indexed256 | TrueColor
    bool:synchronized;         // DEC 2026 — wrap each frame
    bool:bracketed_paste;      // DEC 2004
    bool:focus_events;         // DEC 1004
    bool:mouse_sgr;            // DEC 1006
    bool:mouse_sgr_pixel;      // DEC 1016
    bool:kitty_keyboard;       // CSI > flags u
    uint8:kitty_flags;         // what we push if we push
    bool:modify_other_keys;    // CSI > 4 ; 2 m
    bool:underline_styles;     // SGR 4:1 .. 4:5
    bool:underline_color;      // SGR 58 / 59
    bool:hyperlinks;           // OSC 8
    bool:title_stack;          // CSI 22/23 ; 2 t
    bool:cursor_style;         // DECSCUSR
    bool:sixel;                // recorded, not used until an image cycle
    bool:unicode_core;         // DEC 2027, if the terminal claims mode-2027 width
    // width behaviour, measured (TEXT_MODEL.md §4)
    bool:zwj_composes;
    bool:ri_pairs;
    bool:vs16_widens;
    bool:cjk_wide;
    bool:ambiguous_wide;
    // provenance, so a diagnostic can say WHY
    CapSource:source_of_color;
    string:term_name;          // as identified, e.g. "kitty 0.32.2"
    bool:multiplexed;          // inside tmux or screen
};

pub enum:ColorDepth = { None; Ansi16; Indexed256; TrueColor; };
pub enum:CapSource  = { Table; Environment; Probe; Application; };
```

**Rule C-1.** `Caps` is a plain value with no owning fields, resolved **once**
at `ntui_enter()` and never mutated afterwards except by an explicit
application override before the first frame. Nothing consults the environment
or the terminal during rendering. This is what makes a frame's bytes a function
of `(front, back, caps)` and therefore reproducible in a test.

---

## 3. Resolution order

Later sources win over earlier ones. Each step records its `CapSource` for the
fields it set, so `ntui_caps_explain()` can print a line per capability saying
where the answer came from — which is the difference between a five-minute and
a five-hour investigation when a terminal misbehaves.

**1. The built-in table**, keyed by `$TERM_PROGRAM` first and `$TERM` second.
Generated into `src/caps/table.npk` from `tools/caps.toml` by
`tools/gen_caps.py`, checked by regeneration exactly as the Unicode tables are
(TEXT_MODEL.md §5). The default row — for a `$TERM` nobody recognises that
contains `256color` or `color` — is `Indexed256`, SGR mouse, bracketed paste,
and nothing else.

**2. The environment**, in this order:

| Variable | Effect |
|---|---|
| `NO_COLOR` (set, non-empty) | `color = None`. Absolute; nothing later re-enables it. |
| `CLICOLOR_FORCE` (set, non-empty, not `0`) | colour stays on even where a table row said otherwise. Never overrides `NO_COLOR`. |
| `COLORTERM` = `truecolor` \| `24bit` | `color = TrueColor` |
| `TERM_PROGRAM`, `TERM_PROGRAM_VERSION` | refine the table row |
| `VTE_VERSION`, `KONSOLE_VERSION`, `WT_SESSION`, `TERMUX_VERSION`, `LC_TERMINAL` | identify a terminal whose `$TERM` is generic |
| `TMUX`, `STY`, `$TERM` starting `screen` or `tmux` | `multiplexed = true` — see §6 |
| `NTUI_CAPS` | the escape hatch: a comma-separated list of `name` / `no-name` that overrides everything, for a user debugging their own terminal |

**3. The runtime probe** (§4), which is authoritative for everything it can
answer.

**4. The application**, which may set any field before the first frame and
whose value is final. `CapSource.Application` is recorded so a bug report says
so.

**Rule C-2.** `NO_COLOR` is honoured at the top of the ladder and is not
overridable by the probe. A user who has asked for no colour has asked for no
colour, and a terminal reporting that it supports 16 million of them is not an
argument.

---

## 4. The probe

**Rule C-3.** One round trip, one write, bounded by `NTUI_PROBE_TIMEOUT`
(100 ms), issued before the alternate screen is entered so that anything a
non-cooperating terminal echoes lands on the normal screen and scrolls away
rather than corrupting frame one.

The write, in one buffer, in this order — the last element is the **sentinel**:

| # | Sequence | Asks |
|---|---|---|
| 1 | `CSI > 0 q` | XTVERSION — name and version, as `DCS > \| <text> ST` |
| 2 | `CSI ? u` | kitty keyboard flags — replies `CSI ? <flags> u`, or nothing |
| 3 | `CSI ? 2026 $ p` | DECRQM: synchronized output |
| 4 | `CSI ? 2004 $ p` | DECRQM: bracketed paste |
| 5 | `CSI ? 1004 $ p` | DECRQM: focus events |
| 6 | `CSI ? 1016 $ p` | DECRQM: SGR-pixel mouse |
| 7 | `CSI ? 7 $ p` | DECRQM: **autowrap, to record its original** (TERMINAL_MODEL.md §7) |
| 8 | `CSI ? 2027 $ p` | DECRQM: mode-2027 grapheme clustering |
| 9 | `CSI c` | **DA1 — the sentinel** |

**Rule C-4 — DA1 is the sentinel and the whole design rests on it.** Every
terminal answers Primary DA. A terminal that does not recognise queries 1–8
answers none of them and answers 9, so the arrival of the DA1 reply means *all
the other answers that were going to come, have come*. There is no per-query
timeout, no heuristic, and no guessing: absent means unsupported.

**Rule C-5.** A DECRQM reply of `CSI ? Ps ; Pm $ y` is read as: `Pm` 1 or 3 →
supported and currently set; 2 or 4 → supported and currently reset; 0 → **not
recognised**, so the capability is false. `Pm` 3 and 4 mean permanently set and
permanently reset, and both count as supported.

**Rule C-6 — the truecolour probe is separate and optional.** There is no query
for it. The reliable in-band test is: set `SGR 38 ; 2 ; 1 ; 2 ; 3 m`, ask
`DECRQSS` for the current SGR (`DCS $ q m ST`), and see whether the reply
echoes the RGB triple back. `ntui` runs it only when the table and the
environment disagree or when both are silent, because it perturbs the SGR
state and one more round trip at startup is not free. Its result is recorded
as `CapSource.Probe`.

**Rule C-7 — replies are events.** The probe does not read the descriptor
directly. It writes, then consumes events from the ordinary input decoder until
the DA1 event or the deadline. This matters: a user who typed while the program
was starting has bytes in the buffer, and those keystrokes are **queued as
input events**, not discarded. A terminal library that eats the first keypress
is one everybody notices.

**Rule C-8 — the probe never runs on the headless device.** `Caps` there is
whatever the test constructed, and `CapSource` is `Application` for everything.

---

## 5. Width calibration

**Rule C-9.** Immediately after the capability probe and before the alternate
screen, `ntui` measures four width behaviours by writing a probe string and
asking `CSI 6 n` where the cursor landed. It is one additional round trip
inside the same deadline.

The probes, each written at a known column with the cursor homed first:

| Probe | Text | Composing terminal | Non-composing |
|---|---|---|---|
| ZWJ | `U+1F468 U+200D U+1F469` | 2 | 4 |
| Regional indicator | `U+1F1EF U+1F1F5` (JP) | 2 | 4 |
| VS16 | `U+2764 U+FE0F` | 2 | 1 |
| CJK | `U+4E00` | 2 | 2 (a disagreement here means the terminal is unusual and everything is suspect) |

**Rule C-10.** Calibration output goes to the **normal** screen before the
alternate screen is entered, and is erased with `CR` + `CSI 2 K` afterwards. A
terminal that fails to answer leaves the static rules in place — no calibration
is a defined state, not a failure.

**Rule C-11.** `Caps.calibrated` is not a field, deliberately: the four
booleans have their own `CapSource`, and "did calibration run" is answered by
looking at them. One fact, one place.

---

## 6. Multiplexers

**Rule C-12.** Inside `tmux` or GNU `screen`, the capabilities that matter are
the **multiplexer's**, not the outer terminal's, and the probe correctly
measures the multiplexer because the multiplexer is what answers. That is the
right answer and needs no special case.

What does need one: `ntui` sets `multiplexed = true` and refuses to *push*
kitty keyboard flags there unless the probe positively confirmed support, and
does not attempt passthrough (`DCS tmux; …`). Passthrough asks a program to
know what its outer terminal is, through a wrapper that may or may not forward
it, and getting it wrong writes garbage to the user's screen. An application
that needs it can emit its own.

---

## 7. Degradation

**Rule C-13.** Degradation is a **total function of `(requested, caps)`
computed once**, not a branch at each write:

- `TrueColor` requested on an `Indexed256` terminal → nearest of the 256-colour
  cube by the fixed mapping in [`STYLE_MODEL.md`](STYLE_MODEL.md) §4.
- `Indexed256` on `Ansi16` → the fixed 256→16 table, also in STYLE_MODEL.
- Anything on `None` → colour is dropped; attributes remain.
- An unsupported attribute is **dropped**, never approximated. An underline
  style the terminal lacks becomes a plain underline; a plain underline it
  lacks becomes nothing. It never becomes reverse video, because a substitution
  that changes what the user sees into something else is worse than the absence.
- An unsupported hyperlink drops the OSC 8 wrapper and keeps the text.

**Rule C-14.** Degradation happens **in the renderer**, at the point of
emission, from the resolved `Caps` — never in the widget, never in the style
value. A `Style` always means the colour the author asked for; what reached the
terminal is a property of that terminal and is visible in the emitted bytes,
which is where a test can see it.

---

## 8. Open items

- **O-C1 — the initial capability table's contents.** **Open by design:** the
  rows are *data*, written at cycle 0.4 against a real matrix (`COMPAT.md`) and
  completed at 0.16. Not a design question and not settleable in advance.
- ~~**O-C2 — whether to probe DECRQM for `?1049`.**~~ — **SETTLED (T-107): no
  probe.** The alternate screen is universal enough that the query buys
  nothing, and a terminal without it ignores both the enter and the leave.
