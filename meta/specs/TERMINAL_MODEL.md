# The terminal device

What `ntui` opens, how it configures it, how it learns the size, how it hears
about signals, and — the part that matters most — how it gives the terminal
back. [`SAFETY.md`](SAFETY.md) §2 states the restore contract; this document
states the mechanism.

Everything here is Linux on x86-64, through the language's `sys` builtin and
the runtime floor's `open` / `read` / `write` / `close`. There is no libc call
anywhere and no C structure crosses any boundary — the kernel's structures are
laid out in `buffer` storage by this library, byte offset by byte offset, and
every offset in this document is a number the test suite asserts.

---

## 1. The value

```nitpick
pub struct:Terminal = {
    OwnedFd:tty;            // the device — closed by its own drop
    OwnedFd:sigfd;          // signalfd, §5
    TtySource:source;       // where the descriptor came from, §2
    uint16:rows;
    uint16:cols;
    uint16:xpixels;         // 0 when the terminal does not report pixels
    uint16:ypixels;
    Caps:caps;              // resolved once, CAPABILITIES.md
    int32:slot;             // this terminal's restore-record slot, §4
    Deadlines:within;       // SAFETY.md §4's three, overridable at construction
};

pub enum:TtySource = { DevTty; Inherited; Headless; };
```

A `Terminal` **owns** its descriptors, so it is move-only and its drop closes
them (D-185: the drop *is* the close). Its drop also runs the restore, which is
[`SAFETY.md`](SAFETY.md)'s rule S-5.

**Rule D-1.** There is exactly one live `Terminal` per process. A second
`ntui_enter()` while one is live fails `ETuiState`. Two objects driving one
terminal is a class of bug with no useful behaviour, and the restore record
(§4) is single-slot for the same reason.

---

## 2. Acquiring the device

**Rule D-2 — `/dev/tty` first.** `ntui_enter()` opens `/dev/tty` with
`O_RDWR | O_NOCTTY | O_NONBLOCK | O_CLOEXEC`. This is the primary path and the
only one that is fully asynchronous.

Three properties follow, and each is a reason rather than a convenience:

- **It is a new open-file description.** Setting `O_NONBLOCK` on it changes
  nothing for any other process. Setting `O_NONBLOCK` on an *inherited*
  descriptor rides the shared description into the parent's shell, which is
  exactly why `IO_REFERENCE.md` §8 makes descriptors 0/1/2 the one stated
  exception to D-071 and leaves them blocking. We do not touch them.
- **It survives redirection.** `myapp < input.txt > out.log` still has a
  controlling terminal, and a TUI that refuses because stdin is a file is a TUI
  that cannot be scripted.
- **`O_NOCTTY` is required**, not decorative: without it a process with no
  controlling terminal would acquire one by opening the device, which is a side
  effect nobody asked for.

**Rule D-3 — the inherited fallback, explicitly.** If `/dev/tty` cannot be
opened (typically `ENXIO`: the session has no controlling terminal), `ntui`
tries `2`, then `1`, then `0`, and takes the first for which the terminal probe
(§3) succeeds. That descriptor is **duplicated** with `F_DUPFD_CLOEXEC` so the
`Terminal`'s drop closes a copy and never the caller's stream, and
**`O_NONBLOCK` is not set on it** — D-071's exception applies and this library
does not get to widen it.

The consequence is recorded in the value: `source == TtySource.Inherited` means

- reads are gated on `EPOLLIN` from the reactor and only issued once readiness
  has been signalled, so a read returns what is there rather than parking, and
- a write to a stuffed terminal **blocks the thread**, bounded by the consumer,
  exactly as `IO_REFERENCE.md` §8 accepts for `std_out`.

**Rule D-4.** If neither path yields a terminal, `ntui_enter()` fails
`ETuiNotATerminal`. It does not guess a size and pretend, because a program
drawing a frame into a pipe is a program producing garbage that looks like
output.

**Rule D-5 — the headless device.** `ntui_headless(rows, cols, caps)` builds a
`Terminal` with `source == Headless` whose "descriptor" is a byte sink and
whose input is a byte source the test supplies. Everything above the device
layer is identical. This is not a testing afterthought bolted on later; it is
the reason the device layer is a separate module, and [`TESTING.md`](TESTING.md)
is built on it.

### 2.1 The terminal probe

**Rule D-6.** "Is this a terminal" is `ioctl(fd, TCGETS, &t)` succeeding, and
nothing else. There is no `isatty` in the language and none is needed: the call
that answers the question is the call that fetches the state we are about to
save.

```
ioctl  = syscall 16
TCGETS = 0x5401
```

---

## 3. termios

**Rule D-7 — the kernel's layout, not glibc's.** The structure `TCGETS` fills
is the kernel's `struct termios`, with `NCCS == 19`:

| Offset | Size | Field |
|---|---|---|
| 0 | 4 | `c_iflag` |
| 4 | 4 | `c_oflag` |
| 8 | 4 | `c_cflag` |
| 12 | 4 | `c_lflag` |
| 16 | 1 | `c_line` |
| 17 | 19 | `c_cc[19]` |

**36 bytes, alignment 4.** glibc's `struct termios` is 60 bytes with `NCCS ==
32` and two trailing speed words, and glibc translates between the two. We talk
to the kernel, so 36 is the number, and a test asserts it by round-tripping a
`TCGETS` into a `TCSETS` and reading it back unchanged.

**Rule D-8 — the ioctls used, and only these.**

| Name | Value | Used for |
|---|---|---|
| `TCGETS` | 0x5401 | probe (§2.1) and capture the original |
| `TCSETS` | 0x5402 | apply, immediately, without draining output |
| `TCFLSH` | 0x540B | discard unread input, when discarding is what is wanted |
| `TIOCGWINSZ` | 0x5413 | the window size (§6) |

**`TCSETSW` and `TCSETSF` are deliberately unused.** Both wait for pending
output to drain, and that wait is unbounded and not interruptible by our
deadline machinery. A restore that can hang is worse than a restore that races
a few queued bytes — and on the trap path (`SAFETY.md` S-2) it is
unacceptable outright. Where discarding stale input is wanted, `TCFLSH` with
`TCIFLUSH` (0) does that on its own, without touching output.

**Rule D-9 — the raw configuration is exact and is stated here.** `ntui` clears
and sets these bits and no others. The value written is the *captured original*
with these edits applied, never a constructed "sane" state — a terminal may
legitimately arrive with unusual speeds, an unusual line discipline, or
non-default control characters, and inventing a replacement discards
information we were handed.

```
c_iflag &= ~(IGNBRK | BRKINT | PARMRK | ISTRIP | INLCR | IGNCR | ICRNL | IXON)
c_oflag &= ~(OPOST)
c_lflag &= ~(ECHO | ECHONL | ICANON | ISIG | IEXTEN)
c_cflag &= ~(CSIZE | PARENB)
c_cflag |=  (CS8)
c_cc[VMIN]  = 1
c_cc[VTIME] = 0
```

with the values, which are constants in `ntui/term/bits.npk` and are asserted
against a running terminal by the device suite:

| Flag | Word | Value | Cleared because |
|---|---|---|---|
| `IGNBRK` | `c_iflag` | 1 | a break is an input event, not something to drop |
| `BRKINT` | `c_iflag` | 2 | a break must not raise `SIGINT` |
| `PARMRK` | `c_iflag` | 8 | parity errors must not be rewritten into the byte stream as `\377\0` |
| `ISTRIP` | `c_iflag` | 32 | the eighth bit is UTF-8, not parity |
| `INLCR` | `c_iflag` | 64 | no translation on input, ever |
| `IGNCR` | `c_iflag` | 128 | `\r` is a byte the decoder needs to see |
| `ICRNL` | `c_iflag` | 256 | `\r` must not arrive as `\n`: Enter and Ctrl-J are different keys |
| `IXON` | `c_iflag` | 1024 | Ctrl-S and Ctrl-Q are key presses, not flow control |
| `OPOST` | `c_oflag` | 1 | no output post-processing: `\n` must not become `\r\n` mid-escape-sequence |
| `ECHO` | `c_lflag` | 8 | the application draws; the kernel does not |
| `ECHONL` | `c_lflag` | 64 | ditto |
| `ICANON` | `c_lflag` | 2 | bytes arrive as they are typed, not by the line |
| `ISIG` | `c_lflag` | 1 | Ctrl-C, Ctrl-\\ and Ctrl-Z are key presses; §5 says what happens to the signals |
| `IEXTEN` | `c_lflag` | 32768 | Ctrl-V must not be swallowed as a literal-next prefix |
| `CSIZE`/`CS8` | `c_cflag` | 48 / 48 | eight-bit characters |
| `PARENB` | `c_cflag` | 256 | no parity |

`c_cc` indices: `VMIN = 6`, `VTIME = 5`.

> **`IUTF8` (`c_iflag`, 16384) is left exactly as found.** It affects the
> kernel's canonical-mode erase handling, which does not run in raw mode, so
> setting it would be a change with no effect — and `ntui`'s rule is to change
> what it needs and nothing else.

**Rule D-10 — `ISIG` off means Ctrl-C is a key.** With `ISIG` cleared, the
kernel raises no signal for `INTR`, `QUIT` or `SUSP`: Ctrl-C arrives as byte
`0x03`, Ctrl-\\ as `0x1C`, Ctrl-Z as `0x1A`, and it is the application that
decides what they mean. This is what a TUI wants and it is what every TUI
library does. It has one consequence that must be said out loud: **an
application that never handles Ctrl-C cannot be interrupted from the
keyboard.** `ntui`'s runtime therefore delivers `Key.Char('c')` with
`Mods.Ctrl` like any other key, and the `App` template in the examples handles
it, but the library does not force a quit — a control loop with actuators live
does not get to be killed by a keystroke without saying so.

Signals raised from *outside* — `kill -TERM`, `kill -TSTP`, the window manager's
`SIGHUP` — still arrive, and §5 is how.

---

## 4. The restore record

**Rule D-11.** One process-wide, statically allocated record:

```nitpick
// Module state, `fixed`-initialised, never reallocated (D-211: module state is
// `fixed`; the mutable interior is a wild-free fixed-size block reached through
// an atomic, exactly as the runtime's own registries are).
struct:RestoreRecord = {
    atomic<int32>:live;         // 0 = nothing to restore, 1 = armed
    int32:fd;                   // the raw descriptor number
    uint8[36]:saved_termios;    // the captured original, byte for byte
    uint32:modes;               // the bitmask of what we turned on
    uint8:orig_decawm;          // 0 unknown, 1 was set, 2 was reset (DECRQM, §7)
};
```

**Rule D-12.** `ntui_restore()` is:

1. `compare_exchange(live, 1, 0)` — if it was not 1, return; this is S-3's
   idempotence and it is atomic, so a `failsafe` racing a drop restores once.
2. Build the exit byte string into a **fixed 256-byte static buffer** from
   `modes` (§7's table, in reverse order). No allocation.
3. `write(fd, …)` in a loop bounded by `mono_now()` against a 100 ms budget,
   retrying `EAGAIN` and `EINTR`, abandoning the remainder if the budget
   expires. `mono_now()` is a floor builtin, `never fails`, and allocates
   nothing — which is what makes a deadline expressible on the trap path at
   all.
4. `ioctl(fd, TCSETS, saved_termios)`.
5. `ioctl(fd, TCFLSH, TCIFLUSH)` — discard input fragments the shell would
   otherwise interpret.
6. Return `NIL`. Every step's failure is ignored: there is no caller who could
   act on it, and a restore that fails half way has still done more good than
   one that gave up at the first error.

The order is **write first, termios second**. The exit sequence is composed for
the mode the terminal is in; restoring `OPOST` before writing it would let
`ONLCR` rewrite bytes inside an escape sequence.

**Rule D-13.** The record is written **before** the first mode change and
cleared **after** the last one, so the window in which the terminal is modified
and unregistered is empty. `ntui_enter()` therefore captures termios, writes
the record with `modes == 0` and `live = 1`, and only then begins issuing mode
changes — updating `modes` after each one succeeds.

---

## 5. Signals

**Rule D-14.** `ntui` installs no signal handler. Signals are **blocked** and
read as bytes from a `signalfd`, which puts them in the epoll reactor beside
the terminal and makes every async-signal-safety question disappear.

```
rt_sigprocmask = syscall 14   (how, set, oldset, 8)      SIG_BLOCK = 0, SIG_SETMASK = 2
signalfd4      = syscall 289  (fd, mask, 8, flags)       SFD_CLOEXEC = 524288, SFD_NONBLOCK = 2048
```

The `sigset_t` the kernel takes here is **8 bytes** (`_NSIG / 8`), little-endian,
bit `n-1` for signal `n`.

**Rule D-15 — the set, exactly.**

| Signal | # | Blocked and watched because |
|---|---|---|
| `SIGWINCH` | 28 | the window was resized; §6 |
| `SIGTERM` | 15 | orderly shutdown — without this, `kill` leaves the terminal wrecked |
| `SIGHUP` | 1 | the terminal went away; shut down without writing to a dead device |
| `SIGINT` | 2 | not raised by the keyboard in raw mode (D-10), but `kill -INT` still arrives |
| `SIGQUIT` | 3 | as `SIGINT` |
| `SIGTSTP` | 20 | an external suspend request; §8 |
| `SIGCONT` | 18 | resumed; §8 |

`SIGPIPE` (13) is **not** in the set and is **not** blocked: the language's
floor already returns `EPIPE` in `Result.err`, and a write to a closed terminal
is an error the caller handles, not a signal. Nothing else is touched.

**Rule D-16.** The previous mask is saved in the restore record's neighbourhood
and restored by `ntui_restore()`'s caller on the normal path
(`rt_sigprocmask(SIG_SETMASK, saved, …)`). It is **not** restored on the trap
path: the process is about to exit, the mask dies with it, and one more syscall
in `failsafe` buys nothing.

**Rule D-17.** A `signalfd` read yields fixed 128-byte `signalfd_siginfo`
records; `ssi_signo` is the `uint32` at offset 0 and is the only field `ntui`
reads. The descriptor is `SFD_NONBLOCK`, so a read drains everything pending
and returns `EAGAIN`; several `SIGWINCH`es coalesce into one re-query, which is
correct — the size is whatever it is *now*.

**Rule D-18 — the mask is inherited across `execve`.** A child spawned while
`ntui` holds the terminal inherits the blocked set and will not receive
`SIGWINCH`, `SIGTERM` or `SIGTSTP` — which for an editor or a pager is a
serious misbehaviour. The floor's `clone_exec` primitive runs a fixed
allocation-free child sequence (PDEATHSIG, the parent check, `NO_NEW_PRIVS`,
the `dup3` shuffle, `execve`) with **no slot for a signal mask**, so the child
cannot clear it itself. `ntui_suspend()` (§8) therefore unblocks before the
caller spawns and `ntui_resume()` re-blocks after, and the size is re-queried
on return because a `SIGWINCH` during the window is lost. This is recorded as a
compiler-side gap in [`../OPEN_QUESTIONS.md`](../OPEN_QUESTIONS.md) (O-N1): a
`sigmask` word in `clone_exec`'s parameter block would close it properly.

---

## 6. The window size

**Rule D-19 — the ioctl is the answer.** `TIOCGWINSZ` (0x5413) fills

| Offset | Size | Field |
|---|---|---|
| 0 | 2 | `ws_row` |
| 2 | 2 | `ws_col` |
| 4 | 2 | `ws_xpixel` |
| 6 | 2 | `ws_ypixel` |

**8 bytes.** `ws_xpixel` / `ws_ypixel` are zero on terminals that do not report
them, and zero means "unknown" — never "zero pixels". They are carried because
the image protocols and the sixel/braille scaling need them, and because a
value we already have costs nothing to keep.

**Rule D-20 (T-015) — the ladder, in order, no silent default.**

1. `TIOCGWINSZ`, and if `ws_row > 0 && ws_col > 0` that is the size.
2. `$LINES` / `$COLUMNS` from `environ()`, if both parse to a positive value.
   Some environments — a `screen` session detached and reattached, an
   emulator's first frame — genuinely have a zero ioctl and a correct
   environment.
3. The in-band query `CSI 18 t`, whose reply is `CSI 8 ; rows ; cols t`,
   bounded by `NTUI_PROBE_TIMEOUT`. This is the fallback and never the primary:
   it needs the input stream, it needs the terminal to cooperate, and its reply
   arrives interleaved with user input.
4. Fail `ETuiSize`. **There is no 80×24 default.** A frame drawn at a guessed
   size onto a terminal of another size is worse than a refusal, and the
   application is in a far better position to decide whether to proceed.

**Rule D-21.** The size is re-read by step 1 alone whenever a `SIGWINCH` is
drained. A resize is a `Event.Resize(rows, cols)` on the event stream, and the
runtime marks the whole screen damaged (SCREEN_MODEL.md §7).

---

## 7. Modes, and the exit sequence

**Rule D-22.** Every mode `ntui` can turn on has a bit in `modes`, an entry
sequence, and an exit sequence, and the exit sequence is emitted **in reverse
bit order**. This table is the authority; the restore code is generated from
it by a test that reconstructs the string and compares.

| Bit | Mode | Enter | Exit |
|---|---|---|---|
| 0 | alternate screen | `CSI ? 1049 h` | `CSI ? 1049 l` |
| 1 | cursor hidden | `CSI ? 25 l` | `CSI ? 25 h` |
| 2 | autowrap off | `CSI ? 7 l` | `CSI ? 7 h` or `CSI ? 7 l`, per `orig_decawm` |
| 3 | bracketed paste | `CSI ? 2004 h` | `CSI ? 2004 l` |
| 4 | focus reporting | `CSI ? 1004 h` | `CSI ? 1004 l` |
| 5 | mouse: buttons | `CSI ? 1000 h` | `CSI ? 1000 l` |
| 6 | mouse: drag | `CSI ? 1002 h` | `CSI ? 1002 l` |
| 7 | mouse: any motion | `CSI ? 1003 h` | `CSI ? 1003 l` |
| 8 | mouse: SGR encoding | `CSI ? 1006 h` | `CSI ? 1006 l` |
| 9 | mouse: SGR pixel | `CSI ? 1016 h` | `CSI ? 1016 l` |
| 10 | kitty keyboard | `CSI > <flags> u` | `CSI < 1 u` (pop) |
| 11 | `modifyOtherKeys` | `CSI > 4 ; 2 m` | `CSI > 4 ; 0 m` |
| 12 | title pushed | `CSI 22 ; 2 t` | `CSI 23 ; 2 t` |
| 13 | cursor style set | `CSI <n> SP q` | `CSI 0 SP q` |
| 14 | scrolling region set | `CSI <t> ; <b> r` | `CSI r` |

Plus, unconditionally on exit and not a mode: `CSI 0 m` (reset attributes),
emitted after the mouse and keyboard modes and before showing the cursor, so a
terminal that renders during the sequence never renders styled.

> **Autowrap is turned off** (bit 2) because the bottom-right cell is
> unwritable with it on — the terminal scrolls. Turning it off makes the whole
> grid addressable and removes the special case every other library carries.
> Its original state is read by `DECRQM ? 7` during the capability probe and
> restored to what it was, not to the default, because `SAFETY.md` S-2's rule
> is that we restore what we changed.

**Rule D-23.** The kitty keyboard flags are **pushed**, never set: `CSI > f u`
pushes onto the terminal's own stack and `CSI < 1 u` pops exactly our entry,
so an application running inside another that had its own flags set does not
clobber them. `CSI = f ; m u` (set) is not used.

---

## 8. Suspend and resume

**Rule D-24 (T-016).** `ntui_suspend()` and `ntui_resume()` are public, and are what
an application calls around anything that takes the terminal away — a spawned
`$EDITOR`, a pager, a shell escape, or its own handling of Ctrl-Z.

`ntui_suspend()`:

1. runs the restore (§4), leaving the terminal exactly as `ntui` found it, and
   clears the record;
2. `rt_sigprocmask(SIG_SETMASK, saved_mask, …)` — unblocks, so a child spawned
   next inherits a clean mask (D-18);
3. leaves the `Terminal` value alive, holding its descriptors, in a state where
   any draw call fails `ETuiState`.

`ntui_resume()` reverses it: re-block, re-capture termios (the child may have
changed it), re-arm the record, re-issue every mode in `modes`, re-query the
size by `TIOCGWINSZ`, and emit an `Event.Resize` unconditionally — the size may
have changed while we were not listening, and one redundant full repaint is
cheaper than one missed one.

**Rule D-25 — the external `SIGTSTP`.** Reading `SIGTSTP` from the signal fd
means someone outside asked us to stop. The runtime then: suspends (D-24),
unblocks `SIGTSTP`, raises it at itself (`kill(getpid(), SIGSTOP)` — `SIGSTOP`,
not `SIGTSTP`, because our own handler-free process would otherwise just queue
it again), and on return from stop — which is where `SIGCONT` puts us —
re-blocks and resumes. The `SIGCONT` record that then appears on the signal fd
is drained and produces no event, because the resume already happened.

> Ctrl-Z from the keyboard does **not** reach here: with `ISIG` off it is byte
> `0x1A`, a `Key.Char('z')` with `Mods.Ctrl`, and an application that wants
> Ctrl-Z to suspend calls `ntui_suspend()` and the sequence above itself. The
> two paths meet in one function.

---

## 9. Reading and writing

**Rule D-26 — one read path.** Input is read into a fixed 4096-byte staging
buffer owned by the `Terminal`, and handed to the decoder as a `uint8[]` slice.
The decoder is a pure function (INPUT_MODEL.md §2), so nothing about the device
reaches it. On `source == DevTty` the read is `read`-then-`io_ready`-retry, the
prelude's own `ByteReader` shape; on `Inherited` it is `io_ready`-then-`read`,
because the descriptor is blocking and readiness must precede the call.

**Rule D-27 — one write path, one write per frame.** The renderer composes a
whole frame into an owned `buffer` and the device issues it with a single
`write` loop (retrying short counts, `EAGAIN` and `EINTR`, bounded by
`NTUI_WRITE_TIMEOUT`). Never a write per cell, never a write per widget. This
is what makes a frame atomic enough to be worth wrapping in synchronized output
(SCREEN_MODEL.md §8), and it is what makes the byte stream assertable in a test.

**Rule D-28.** The device never writes anything the renderer did not ask for.
There is no implicit flush, no implicit clear, no implicit cursor move. The one
exception is the restore's exit sequence, which is the device's own and is
specified in §7.

---

## 10. Open items

- **O-T1 — `TIOCSWINSZ` for the headless device.** Not needed while the
  headless device is a pure value, but if a future integration test drives a
  real pty pair, setting the size on the master is how a resize is simulated.
  Deferred until such a test exists.
- **O-T2 — the pty pair as a test device.** `openpt`-equivalent via
  `/dev/ptmx` would let the suite drive a *real* line discipline rather than a
  headless sink. It is strictly more faithful and strictly more machinery;
  recommendation is to add it at the hardening cycle if the headless oracle has
  by then been shown to miss something, and not before.
