# Safety, errors, and the `failsafe` contract

The constraints. Read this first: most designs that look reasonable for a TUI
library die on one of §1's rules, and the two that shape `ntui` most —
the trap path and the error budget — have no analogue in any other language's
TUI library, so there is nothing to copy.

---

## 1. What the language imposes

Each row is a language decision, not ours. The consequence column is what it
costs a library that draws on a terminal.

| Language rule | Where | Consequence for `ntui` |
|---|---|---|
| `defer` does **not** run on a trap | D-014 | Terminal restoration cannot live in a `defer`. §2. |
| A trap is a **whole-program** event; no task resumes | D-063 | There is no "the render task cleans up". §2. |
| Everything uncaught reaches `failsafe`, which then exits | D-013, D-142 | `failsafe` is the only code guaranteed to run. §2. |
| `failsafe`'s `pick` must **name** every error that can reach it | REACH-002 | Every public `error:` we declare is an arm every consuming program owes. §3. |
| Reachability is **import-scoped** | 1.4.8's `nsys` note | Module decomposition decides what a consumer's `failsafe` owes. §3. |
| Plain integer `+ - *` **traps** on overflow | D-210 | Layout arithmetic must not rely on wrapping; widening is explicit. §5. |
| `/` and `%` by zero trap; signed `MIN / -1` traps | D-007, D-142 | A divisor is checked or proven. §5, VERIFICATION.md. |
| Array, slice and buffer indexing is bounds-checked and traps | D-070 | An out-of-range cell index is a *crash*, not a smear. §5. |
| Owning values are **move-only** | TYPE-046 | No binding-to-binding copies of a `string`, `buffer`, `OwnedFd`. §6. |
| Borrows are second class | D-004 | A cell view cannot be returned, stored, sent, or held across `await`. §6. |
| A successful `exit 0` with live `wild` allocations **traps** | D-151 | Every `wild` byte we take is paired or we cannot exit clean. §6. |
| A clean exit with a registered child **traps** | D-188 | A spawned editor/pager must be reaped. §6. |
| There are **no closures** | D-018 | Callbacks are values the runtime interprets. EVENT_MODEL.md. |
| There is **no `select`** across channels | D-072 | Fan-in is one channel, or one `suspend_io` over several watched descriptors. EVENT_MODEL.md. |
| Every blocking operation carries a **mandatory deadline** | D-056, D-176 | There is no unbounded read on the terminal. §4. |
| Blocking is **task** suspension, never thread parking | D-071 | Terminal I/O goes through the reactor. TERMINAL_MODEL.md §6. |
| Inherited descriptors 0/1/2 stay **blocking** — the one stated exception | D-071, D-185 | We must not set `O_NONBLOCK` on them. TERMINAL_MODEL.md §2. |
| `async` functions can never be `never fails` | D-163 | `raw await …` is unspellable; `relay` / `?|` / `?!` are the spellings. |

---

## 2. The trap path, and why terminal restoration lives on it

**The failure this exists to prevent.** A program that stops while holding the
terminal leaves it in raw mode with the alternate screen active, the cursor
hidden, mouse reporting on and bracketed paste on. The user's shell then does
not echo, does not line-edit, does not scroll, and prints mouse escape garbage
on every click. The user's recourse is `reset` typed blind. Every TUI library
in every language has this hazard; most answer it with an atexit handler and a
signal handler, and both are defeated by the failure modes that actually
happen — a hard fault, a `SIGKILL`, an abort inside the handler itself.

Nitpick's answer is structural, and `ntui` uses it as given:

**Rule S-1.** Terminal state is recorded in a **restore record**: a single
fixed-size, statically allocated structure holding the captured original
`termios`, the descriptor, and a bitmask of the modes `ntui` has turned on. It
is written before the first mode change and cleared after the last one. It
allocates nothing, ever.

**Rule S-2.** `ntui_restore()` reads the restore record, issues **one**
`ioctl(TCSETSF)` with the captured original `termios` and **one** `write(2)`
carrying the exit sequence for exactly the modes the mask says are on, and
clears the record. It calls no allocator, takes no lock, awaits nothing, and
returns `NIL`. It is safe in a process whose heap is corrupt.

**Rule S-3.** `ntui_restore()` is **idempotent**. Calling it twice restores
once; calling it when nothing was entered does nothing. This is what lets the
same function serve the normal exit path, a `defer`, and `failsafe` without
anyone coordinating.

**Rule S-4.** A program using `ntui` **must** call `ntui_restore()` as the
first statement of its `failsafe`. This is stated in the public documentation,
demonstrated in every example, and checked by the conformance suite; it cannot
be enforced by the compiler, so it is stated in exactly one sentence and
repeated nowhere, so that the one sentence is the thing people learn.

```nitpick
func:failsafe = int32(Error:e) {
    drop ntui_restore();          // S-4: FIRST, before anything else
    pick (e) {
        (ETui)             { exit 70i32; },
        // … the arms REACH-002 requires; §3 lists exactly which
        (*)                { exit 99i32; }
    }
    exit 99i32;
};
```

**Rule S-5.** The normal exit path restores too, through scope-exit drop: the
`Terminal` value owns its `OwnedFd` and its restore-record slot, and its drop
runs `ntui_restore()`. S-4 is the belt for the path where drops do not run;
this is the braces for the path where they do. Neither is a substitute for the
other, and S-3 is what makes having both harmless.

**What S-2 deliberately does not do.** It does not flush a render buffer, does
not clear the screen, does not move the cursor to a chosen row, and does not
print anything of its own. At trap time we do not know what is true; the goal
is a usable terminal, not a tidy one. Anything cosmetic is the application's,
in the arm of its `failsafe` that ran after the restore.

> **A limitation stated rather than mitigated.** `SIGKILL` cannot be caught by
> anything, `signalfd` included, so a `kill -9` leaves the terminal as it was.
> No library on any platform can do better, and pretending otherwise is worse
> than saying so. `ntui`'s answer is the same as `stderr`'s in
> `IO_REFERENCE.md` §4.1: state the residue, do not manufacture a mitigation
> that only appears to work.

---

## 3. The error budget

**Rule S-6.** REACH-002 makes every `error:` that can reach `failsafe` a named
arm the consuming program's `failsafe` must carry. A library that declares
thirty errors makes thirty arms mandatory in every program that imports it —
in a language where forgetting one is a compile error. **The number of public
error identities `ntui` declares is therefore a hard budget, and it is nine.**

| Error | Raised when |
|---|---|
| `ETuiDevice` | the terminal device could not be opened, configured, or restored |
| `ETuiNotATerminal` | the target is not a terminal (the `TCGETS` probe failed) |
| `ETuiSize` | the window size could not be determined by any means |
| `ETuiInput` | the input stream failed, or a decoder invariant was violated |
| `ETuiOutput` | a write to the terminal failed after its retries |
| `ETuiEncoding` | text handed to the library is not valid UTF-8 where validity was required |
| `ETuiGeometry` | a geometry request cannot be satisfied (a rect outside its parent, a constraint set that cannot be met) |
| `ETuiCapacity` | a fixed-capacity structure is full (the event queue, the cluster spill table) |
| `ETuiState` | an API was used out of order (rendering before entering, entering twice) |

**Rule S-7.** Nine is a ceiling, not a target. A new identity is added only by
a recorded decision, and the decision has to say why the failure is one a
`failsafe` would treat differently from all nine. Where the distinction is for
the caller rather than for the shutdown handler, it rides as a **detail field**
on the value the call returns, not as a new error.

**Rule S-8.** Kernel errnos are **forwarded verbatim** where the caller can act
on them — `fail r.err` — exactly as `lib/nsys.npk` does. A forwarded errno is a
dynamic operand and does not enlarge the reachable set, so it costs no
`failsafe` arm. Wrapping an errno in one of the nine is done only where the
errno would tell the caller nothing (`ioctl` failing on a descriptor we opened
ourselves).

**Rule S-9.** Module decomposition is part of the budget, because REACH is
import-scoped. A program that imports only `ntui/text.npk` for width
measurement owes no device arms. The public modules are therefore split so
that:

- `ntui/text.npk` and `ntui/layout.npk` declare **no errors at all** — they are
  pure computation over values, and a failure is a `Result` with a forwarded
  code or an in-band answer.
- `ntui/style.npk` and `ntui/screen.npk` declare `ETuiGeometry`, `ETuiCapacity`
  and `ETuiEncoding` only.
- The device, input and runtime modules declare the remaining six.

**Rule S-10.** The exact arm set a consuming program owes, per import, is
generated into the documentation and checked by a conformance test that builds
a program importing each public module and asserts its `failsafe` compiles with
exactly the documented arms and no more. An out-of-date arm list is the kind of
document that goes stale silently, so it is derived, not written.

---

## 4. Deadlines

**Rule S-11.** Every `ntui` operation that can wait takes a `Duration:within`
and returns `DeadlineExceeded` when it expires. There is no unbounded read, no
unbounded write, and no unbounded probe.

**Rule S-12.** Three deadlines are library-level policy and are named
constants, not magic numbers at call sites:

| Constant | Default | What it bounds |
|---|---|---|
| `NTUI_ESC_TIMEOUT` | 25 ms | how long a lone `ESC` waits for a continuation before being reported as the Escape key (INPUT_MODEL.md §6) |
| `NTUI_PROBE_TIMEOUT` | 100 ms | how long the capability probe waits for the terminal to answer before falling back to the table (CAPABILITIES.md §4) |
| `NTUI_WRITE_TIMEOUT` | 2 s | how long a frame write retries a stuffed terminal before failing `ETuiOutput` |

Each is overridable at `Terminal` construction. The defaults are stated here so
there is one place to change them and one place to argue about them.

**Rule S-13.** A deadline that expires is never silently retried and never
turns into a longer deadline. `DeadlineExceeded` reaches the caller.

---

## 5. Arithmetic

**Rule S-14.** All screen geometry is `uint16`: 0 … 65535 rows and columns. A
terminal larger than that does not exist, the type makes negative coordinates
unspellable, and every product of two of them fits `uint32` with room to spare.

**Rule S-15.** Intermediate arithmetic that could exceed the operand width
**widens explicitly** and narrows with `=>!` at a point where the value is
known to fit — the D-210.3 idiom. A layout computation multiplying a percentage
by an extent computes in `uint32` and narrows once.

**Rule S-16.** Nothing in the library divides by a value it has not proven
non-zero on the same path. Where the proof is not local, the divisor is
checked and the zero case is an answer (an empty split, a zero ratio), not an
error.

**Rule S-17.** Indexing into a cell buffer goes through one accessor pair per
buffer type, and the accessor is where the bound is checked. Callers do not
index raw storage. This is what makes the bound one obligation to discharge in
cycle 1.5 rather than several hundred.

---

## 6. Resources

**Rule S-18.** `ntui` allocates from the **managed** regime. Buffers are
`buffer`; growable storage is the library's own `Vec<T>`
(BUILD.md §5), whose block is `wild` and whose lifetime is its owner's scope.
Every `wild` byte the library takes is released on every path out of the
function that took it, so `exit 0` never trips D-151.

**Rule S-19.** The library holds exactly **three** descriptors at most: the
tty, the signal fd, and (only while a probe is outstanding) nothing else. Each
is an `OwnedFd`, so the close is the drop, and none of them is 0, 1 or 2.

**Rule S-20.** `ntui` spawns no processes. An application that shells out does
so itself, and §7 states what it owes.

**Rule S-21.** `ntui` installs no signal handler, at any point, for any signal.
Signals arrive as bytes on a descriptor (TERMINAL_MODEL.md §5).

---

## 7. What an application owes

Stated once, here, and repeated in the public documentation:

1. Call `ntui_restore()` as the **first** statement of `failsafe` (S-4).
2. Carry the `failsafe` arms your imports require (S-10 generates the list).
3. Around a spawned child that will use the terminal (an editor, a pager),
   call `ntui_suspend()` before and `ntui_resume()` after. The suspend
   restores the terminal and unblocks the signal mask the child would
   otherwise inherit; the resume re-enters and re-queries the size
   (TERMINAL_MODEL.md §8).
4. Reap what you spawn — D-188 traps a clean exit that left a registered child.

---

## 8. Open items

- **O-S1 — the reserved `Error` code range.** `ntui`'s nine identities are
  derived from `module.Name` by the D-179 FNV scheme, so they cannot collide by
  construction, and no range needs reserving. Recorded here because the
  question is asked every time somebody meets the error system, and the answer
  is "the scheme already handles it".
- **O-S2 — whether `ntui_restore()` should also emit `DECSTR` (soft reset).**
  Argument for: it repairs modes `ntui` never set but the application did.
  Argument against: it repairs modes the *user's shell* set deliberately before
  the program ran, and S-2's rule is to restore what we changed and nothing
  else. Recommendation: no, and the escape hatch is that an application may
  emit it from its own `failsafe` arm after the restore. Tracked in
  `../OPEN_QUESTIONS.md`.
