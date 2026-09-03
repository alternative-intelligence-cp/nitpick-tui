# Cycle 0.1 — The device

**`src/term/`: `/dev/tty`, termios, the window size, signals, the restore
record, suspend and resume, and the headless device.** Everything that touches
the kernel, in one module, behind one boundary.

> **`0.1.0.md` is written execution-grade at cycle 0.0's close** (0.0.5, step
> 4), so that this cycle is openable by a session that was not present for the
> probes. That is the convention for every cycle: the opening subcycle file is
> written by the cycle before it.

## Why here

Not because much depends on it — the headless device (T-090) means everything
above it is testable with no terminal at all. **Because it is the riskiest
part of the library and it carries its headline safety property.** If
`/dev/tty`, the restore record on the trap path, or `signalfd` do not work as
`specs/TERMINAL_MODEL.md` says, that is a cycle-0.1 problem, not a cycle-0.10
surprise. Probes 09, 11, 12 and 13 in cycle 0.0 have already asked the
questions; this cycle builds the answers.

## Decisions in

T-010 … T-018, and T-090. All settled.

Plus **T-105** from the second batch: the restore does **not** emit `DECSTR`.

**Open questions to settle in this cycle:** none.

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| 0.1.0 | **The syscall vocabulary** — `src/term/sys.npk`: the numbers, the ioctl constants, the flag words, the structure offsets, each with the source it came from | every constant exercised by a program that asserts its effect, not just its value |
| 0.1.1 | **Acquisition** — `/dev/tty`, the inherited fallback, `TtySource`, the `TCGETS` terminal probe | a `Terminal` value on both paths, with the right descriptor and the right blocking mode |
| 0.1.2 | **The restore record** — the static record, `ntui_restore()`, the mode table and the exit-sequence builder | the trap-path gate: `stty -a` identical before and after a deliberate trap |
| 0.1.3 | **Raw mode** — the exact bit edits, entry and exit, and the `modes` bookkeeping | every bit in `specs/TERMINAL_MODEL.md` §3's table asserted individually |
| 0.1.4 | **The size** — `TIOCGWINSZ`, the environment fallback, the ladder, `ETuiSize` | a size on three real terminals and a refusal where there is none |
| 0.1.5 | **Signals** — the blocked mask, `signalfd4`, the record reader, the event mapping | a self-sent signal of each of the seven arrives as its event |
| 0.1.6 | **Suspend and resume** — `ntui_suspend()`, `ntui_resume()`, the external `SIGTSTP` path | a program that suspends, runs `cat` on the terminal, and resumes with the screen intact |
| 0.1.7 | **The headless device** — `ntui_headless()`, the byte sink and byte source | every 0.1 behaviour that does not need a kernel, exercised headless |
| 0.1.8 | **Close** | `done/0.1/`, `0.2.0.md` written |

## Checklist

### 0.1.0 — the syscall vocabulary
- [ ] `src/term/sys.npk`: `SYS_IOCTL` 16, `SYS_RT_SIGPROCMASK` 14, `SYS_SIGNALFD4` 289, `SYS_EVENTFD2` 290, `SYS_KILL` 62, `SYS_GETPID` 39, `SYS_FCNTL` 72
- [ ] `TCGETS` 0x5401, `TCSETS` 0x5402, `TCFLSH` 0x540B, `TIOCGWINSZ` 0x5413
- [ ] the termios bit constants, every one in `specs/TERMINAL_MODEL.md` §3's table
- [ ] the `c_cc` indices `VMIN` 6, `VTIME` 5
- [ ] `SFD_CLOEXEC` 524288, `SFD_NONBLOCK` 2048, `EFD_CLOEXEC`, `EFD_NONBLOCK`, `SIG_BLOCK` 0, `SIG_SETMASK` 2
- [ ] the seven signal numbers
- [ ] **structure sizes asserted by a program, not by a comment**: termios 36, winsize 8, `signalfd_siginfo` 128, `sigset_t` 8
- [ ] every constant carries the header it came from as a comment, so a reader can check it
- [ ] this module declares **no error of its own** — kernel errnos forward verbatim (`specs/SAFETY.md` S-8)

### 0.1.1 — acquisition
- [ ] `/dev/tty` with `O_RDWR | O_NOCTTY | O_NONBLOCK | O_CLOEXEC`
- [ ] the `TCGETS` probe as the "is this a terminal" test (D-6)
- [ ] the inherited fallback: 2, then 1, then 0; `F_DUPFD_CLOEXEC`; **no `O_NONBLOCK` set** (D-3)
- [ ] `TtySource` recorded in the value and reachable by a caller
- [ ] `ETuiNotATerminal` when neither path yields one — and **no guessed size, no fake terminal**
- [ ] the second `ntui_enter()` while one is live fails `ETuiState` (D-1)
- [ ] a test that runs with stdin, stdout and stderr all redirected to files and still acquires the terminal

### 0.1.2 — the restore record
- [ ] the static record: an `atomic<int32>` liveness word, the descriptor, 36 saved bytes, the mode mask, the original autowrap state
- [ ] `ntui_restore()`: compare-exchange, build into a **fixed 256-byte static buffer**, one bounded `write`, `TCSETS`, `TCFLSH`
- [ ] **no allocation on any path through it** — verified by a test that exhausts the heap and then restores
- [ ] idempotent: called twice, restores once (D-12 step 1)
- [ ] the mode table and the exit-sequence builder, generated from `specs/TERMINAL_MODEL.md` §7 and checked against it by a test that reconstructs the string
- [ ] the write's retry loop bounded by `mono_now()` against 100 ms, abandoning the remainder
- [ ] **the gate**: a program that enters raw mode, sets every mode bit, and traps; a wrapper script captures `stty -a` before and after and requires them equal
- [ ] the same, with the trap inside `failsafe` itself (which exits 70 directly)
- [ ] T-105 honoured: the restore emits no `DECSTR`, and a comment in the code says why

### 0.1.3 — raw mode
- [ ] the edit applied to the **captured original**, never a constructed state (T-014)
- [ ] every bit in §3's table asserted individually: set it, read it back, confirm the behaviour it names
- [ ] `ISIG` off proven: Ctrl-C arrives as byte `0x03` and raises nothing
- [ ] `IXON` off proven: Ctrl-S arrives as `0x13` and does not stop output
- [ ] `OPOST` off proven: a written `\n` emerges as one byte
- [ ] `IUTF8` left exactly as found
- [ ] `modes` updated **after** each mode change succeeds, and the record written **before** the first (D-13)

### 0.1.4 — the size
- [ ] `TIOCGWINSZ` with the four `uint16` fields at 0, 2, 4, 6
- [ ] `ws_xpixel`/`ws_ypixel` carried, with zero meaning unknown
- [ ] the ladder in order: ioctl, `$LINES`/`$COLUMNS`, `CSI 18 t`, then `ETuiSize`
- [ ] **no 80×24 default** — a test asserts the refusal
- [ ] a resize re-reads by the ioctl alone and produces `Event.Resize`

### 0.1.5 — signals
- [ ] the 8-byte sigset built for the seven, little-endian, bit `n-1`
- [ ] `rt_sigprocmask(SIG_BLOCK)`, the old mask saved
- [ ] `signalfd4(-1, mask, 8, SFD_CLOEXEC|SFD_NONBLOCK)`
- [ ] the 128-byte record read, `ssi_signo` at offset 0
- [ ] several pending signals coalesce correctly — a test sends five `SIGWINCH` and expects one re-query
- [ ] `SIGPIPE` **blocked** (T-113), its record drained and discarded, and a test on the **inherited** path that writes to a pipe whose reader has exited and gets `EPIPE` in `Result.err` rather than dying of signal 13
- [ ] the mask restored by `ntui_restore()`'s caller on the normal path and **not** on the trap path (D-16)
- [ ] `// stress: 40` on the signal tests

### 0.1.6 — suspend and resume
- [ ] `ntui_suspend()`: restore, clear the record, `SIG_SETMASK` the saved mask, mark the terminal unusable for drawing
- [ ] `ntui_resume()`: re-block, re-capture termios, re-arm the record, re-issue every mode in `modes`, re-query the size, emit `Event.Resize` unconditionally
- [ ] a draw call between them fails `ETuiState`
- [ ] the external `SIGTSTP` path: suspend, unblock `SIGTSTP`, `kill(getpid(), SIGSTOP)`, resume on return, drain the `SIGCONT` without emitting an event
- [ ] a manual test: suspend, run `$EDITOR` on the terminal, quit it, resume, and the screen is intact
- [ ] the O-N1 workaround documented in the code with its reason and a pointer to the open question

### 0.1.7 — the headless device
- [ ] `ntui_headless(rows, cols, caps)` with a `Bytes` sink and a byte source
- [ ] `TtySource.Headless`, and every device operation that would touch the kernel answered from the value
- [ ] the sink readable by a test; the source writable by a test
- [ ] the probe and calibration skipped entirely (C-8)
- [ ] every 0.1 behaviour that does not need a kernel re-run headless, so the headless device is exercised from the day it exists

### 0.1.8 — close
- [ ] the `device` stage populated and skipping loudly where there is no terminal
- [ ] `check_failsafe_arms` live, with the device module's arm list generated
- [ ] findings written; `0.2.0.md` written execution-grade; archived

## Gate

**A program that enters every mode, traps deliberately, and leaves a terminal
whose `stty -a` is byte-identical to what it was before.** That is the
library's headline safety claim, and this cycle either establishes it or the
claim comes out of the README.

## Watch for

- **Probe 12's verdict decides how the restore record is spelled.** If module
  state with an atomic interior is not reachable from `failsafe`, stop and
  re-plan — do not invent a workaround. The fallback is a request to the
  compiler and the design blocks on it.
- **`fd` is a type, not a name.** So are `buffer` and `raw`. The device module
  is where all three are wanted most.
- **The kernel's `termios` is 36 bytes and glibc's is 60.** Nothing here goes
  through glibc, so 36 is the number, and 0.1.0's assertion is what proves it
  rather than a comment repeating this sentence.
- **`TCSETSW`/`TCSETSF` are never used** (T-017). They drain output, that wait
  is unbounded, and on the trap path it is unacceptable.
