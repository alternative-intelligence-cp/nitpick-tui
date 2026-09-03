# Compatibility

What `ntui` targets, what it is tested against, and what a claim in this
document means.

---

## 1. What a row in the matrix means

**Rule K-1.** A terminal appears in §3's table only when
`tests/fixtures/input/<terminal>/` exists, its cases pass, and its capability
row in `tools/caps.toml` was written from a **measured** probe transcript
rather than from documentation. A terminal with no fixture directory is not
listed, however well it is expected to work.

This is the difference between a compatibility table and a wish list, and the
project's own experience is the argument: a claim nobody can check is a claim
that goes stale silently.

**Rule K-2.** Three tiers, and they say different things:

| Tier | Means |
|---|---|
| **Verified** | a fixture directory, a measured capability row, and the golden suite run against it by hand at least once per release |
| **Expected** | ECMA-48 plus ANSI colour; no fixtures. Should work. Bugs accepted as bugs. |
| **Unsupported** | stated, with the reason |

---

## 2. The floor

**Rule K-3.** `ntui` requires a terminal that speaks:

- ECMA-48 CSI sequences: `CUP`, `CUU`/`CUD`/`CUF`/`CUB`, `ED`, `EL`, `SGR`;
- the DEC private modes `?25` (cursor visibility) and `?1049` (alternate
  screen);
- ANSI 16-colour SGR (`30`–`37`, `40`–`47`, `90`–`97`, `100`–`107`);
- UTF-8 input and output.

Everything else — 256 colours, truecolour, mouse, bracketed paste, focus
events, synchronized output, styled underlines, hyperlinks, the kitty keyboard
protocol — is a capability, is detected, and degrades
(`CAPABILITIES.md` §7).

**Rule K-4 — Linux on x86-64 only, at 1.0.** The device layer is Linux
syscalls: `ioctl` with Linux ioctl numbers, `signalfd4`, `eventfd2`, and the
kernel's 36-byte `termios`. Porting is a real piece of work — a BSD `termios`
is a different structure with different ioctl numbers and no `signalfd` — and
it is a cycle of its own with its own decision, not something to leave
half-done in a `#[cfg]`. `aarch64` Linux is a smaller step (the syscall numbers
differ, the structures do not) and is the first port to consider.

---

## 3. The matrix

Filled as fixtures are captured; the plan's cycle 0.6 fills the first rows and
the release cycle fills the rest. Every cell is measured.

| Terminal | Tier | Colour | Kitty kbd | Sync | SGR mouse | Styled underline | Hyperlink |
|---|---|---|---|---|---|---|---|
| `kitty` | — | | | | | | |
| `foot` | — | | | | | | |
| `wezterm` | — | | | | | | |
| `alacritty` | — | | | | | | |
| `ghostty` | — | | | | | | |
| `xterm` | — | | | | | | |
| `gnome-terminal` / VTE | — | | | | | | |
| `konsole` | — | | | | | | |
| `contour` | — | | | | | | |
| `rio` | — | | | | | | |
| `st` | — | | | | | | |
| `urxvt` | — | | | | | | |
| `tmux` (over each above) | — | | | | | | |
| GNU `screen` | — | | | | | | |
| Linux console (`/dev/tty1`) | — | | | | | | |
| Windows Terminal (WSL) | — | | | | | | |
| `iTerm2` (over SSH) | — | | | | | | |
| VS Code integrated | — | | | | | | |
| `mosh` | — | | | | | | |

**Rule K-5.** `tmux` and `screen` are rows in their own right, not a footnote:
the capabilities inside them are the multiplexer's
(`CAPABILITIES.md` §6), and a program that works in `kitty` and breaks in
`tmux` inside `kitty` is the single most common compatibility report any TUI
library receives.

---

## 4. Unsupported, with reasons

| Not supported | Reason |
|---|---|
| terminfo-described exotic terminals (VT52, Wyse, hardcopy) | `CAPABILITIES.md` §1 — no terminfo, and no fixtures could exist |
| terminals without UTF-8 | the text model is Unicode all the way down; a legacy single-byte encoding would need a transcoding layer nobody has asked for |
| Windows console API (non-VT) | a different device model entirely; the VT mode of Windows Terminal under WSL is a Linux process and is in the matrix |
| right-to-left reordering | `TEXT_MODEL.md` §6 — stated as absent, not attempted badly |
| terminals that require `\n` to advance a line | autowrap is off and the renderer never emits `\n` (`SCREEN_MODEL.md` R-16) |

---

## 5. Degradation, end to end

**Rule K-6.** The suite includes a **degradation matrix test**: one reference
screen rendered under every combination of
`{TrueColor, Indexed256, Ansi16, None}` × `{sync, no sync}` ×
`{kitty kbd, modifyOtherKeys, legacy}` × `{styled underline, plain, none}`,
each with a committed golden. It is a large number of small files and it is
the only way to know that the *combination* nobody runs locally still works.

**Rule K-7.** `NTUI_CAPS` (`CAPABILITIES.md` §3) is how a user reproduces
another terminal's capability set on their own, which makes a bug report
actionable: "run it with `NTUI_CAPS=no-truecolor,no-sync`".
