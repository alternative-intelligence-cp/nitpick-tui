# Open questions

Everything that is not settled, each with a recommendation, so that nothing
lives only in a conversation. Three prefixes:

| Prefix | Whose |
|---|---|
| `O-x` | **ours** — a design question this project decides, at the cycle named |
| `O-N` | the **compiler's** — a gap in the language or its tooling that `ntui` needs closed, to be raised as a request |
| `Q-` | the **user's** — a question that wants an answer before the work it gates begins |

A question that gets answered moves to `DECISIONS.md` as a numbered decision and
is struck through here with the decision's number, **never deleted** — the
question is part of the record of how the answer was reached.

**The second batch (T-100 … T-112) was ratified on 2026-09-03** and struck
thirteen of these. What remains open is what should be: three measurements,
one set of data, one item gated on the compiler's tooling, one waiting for a
consumer to ask, and the four that are the compiler's rather than ours.

---

## Q — for the user

### ~~Q-1 — the Unicode version to pin~~ — **SETTLED, T-100**
The recommendation as written: the latest stable UCD at cycle 0.2, floor
15.1.0, recorded in `src/unicode/version.npk`, and upgrading it is a decision
with a regenerated table set and a re-run golden suite.

### ~~Q-2 — whether `ntui` should ship a `Tree` widget's node store~~ — **SETTLED, T-101**
The trait, over the application's own data. A file browser over a real
directory tree does not have to mirror it into an arena first.

### ~~Q-3 — image protocols (Kitty graphics, Sixel, iTerm2 inline)~~ — **SETTLED, T-102**
Post-1.0, as cycle 1.1, opening with its own decision batch. `Caps.sixel` is
carried from cycle 0.4 so the detection is not re-litigated then.

### ~~Q-4 — the release cadence and versioning~~ — **SETTLED, T-103**
`0.x` until the compiler reaches 1.0 and the API has survived cycle 0.15's
application; semantic versioning thereafter; and **adding a public error
identity is a major version**, because REACH-002 makes it a new mandatory
`failsafe` arm in every consumer — a compiler-enforced source break.

### ~~Q-5 — whether to build a real program as a consumer~~ — **SETTLED, T-104**
Yes: cycle 0.15, a log viewer, in `examples/` in this repository so it moves
with the API. Every friction it meets is recorded and triaged.

---

## O-N — the compiler's

Each is a request to be raised in the compiler repository, not something this
project can fix. None of them blocks planning; two of them block execution.

### O-N1 — `clone_exec` has no signal-mask slot
`ntui` blocks a signal set (`T-011`) and the blocked mask is inherited across
`execve`. The floor's `clone_exec` runs a fixed allocation-free child sequence
with no slot for `rt_sigprocmask`, so a spawned child inherits the block and
will not receive `SIGWINCH`, `SIGTERM` or `SIGTSTP` — a serious misbehaviour
for an editor or a pager. `ntui` works around it by unblocking before the
caller spawns and re-blocking after, losing any signal in the window.
**Ask:** an optional `sigmask` word in `clone_exec`'s ten-word parameter block,
applied in the child before `execve`. Small, and it closes the class.

### O-N2 — `npkg` cannot build a library, and `[dependencies]` resolves to nothing
Measured at the compiler's 1.5.0 and recorded in `specs/BUILD.md` §1.
`npkg build` is the compiler's own bootstrap ladder; `target = "library"` is
accepted by the schema and read by nothing; the loader's dependency-root list
is created empty and `rootlist_add` is called from nowhere, so the
dependency-root `use` form resolves against an empty set.
**Consequence:** `ntui` builds through its own Python harness (T-005) and every
import is relative until this closes.
**Ask:** `npkg build` honouring `target = "library"`, and the driver populating
the resolver's roots from `[dependencies]`. Neither is on the compiler's 1.5 or
1.6 map, so this is a request, not a date.

### O-N3 — no `io_isatty`, as `IO_REFERENCE.md` §4 says there is
The document says "`io_isatty` remains available; it answers a question and
changes nothing", and nothing of that name exists. `ntui` uses the `TCGETS`
probe, which is the same answer by the same mechanism, so this is a
documentation gap rather than a functional one.
**Ask:** either implement it or strike the sentence.

### O-N4 — no timer facility besides `suspend_until`
The runtime's timer wheel (`EVENT_MODEL.md` §5) is built on absolute
`mono_now()` deadlines and the loop's own wait, which is correct and needs
nothing. Recorded only so that a future `timerfd`-shaped addition to the
language is known to have a consumer here.

---

## O-x — ours

### Safety

- **O-S1 — reserved `Error` code range.** Not needed: identities are derived
  from `module.Name` by D-179's FNV scheme and cannot collide. Recorded because
  the question is asked every time somebody meets the error system.
  **Answer: nothing to reserve.**
- ~~**O-S2 — should `ntui_restore()` also emit `DECSTR` (soft reset)?**~~ —
  **SETTLED, T-105: no.** It would repair modes the user's shell set
  deliberately, which is the restore's own rule inverted. An application that
  wants one emits it from its `failsafe` arm after the restore.

### The device

- **O-T1 — `TIOCSWINSZ` for the headless device.** Not needed while the
  headless device is a pure value. If a future integration test drives a real
  pty pair, setting the size on the master is how a resize is simulated.
  **Deferred** until such a test exists.
- **O-T2 — a pty pair as a test device.** `/dev/ptmx` would let the suite drive
  a *real* line discipline rather than a headless sink — strictly more faithful
  and strictly more machinery. **Recommendation:** add it at cycle 0.16 if the
  headless oracle has by then been shown to miss something, and not before.

### Build

- **O-B1 — when to migrate off the harness.** Gated on O-N2. **No action until
  then**; the harness and `npkg` run side by side with a parity check before
  the harness retires, exactly as in the compiler repository.
- ~~**O-B2 — ship as source or as an object.**~~ — **SETTLED, T-112: source.**
  It keeps the closed-world link and whole-program verification intact.
  Revisit only if build times become a real complaint from someone building a
  real program.

### Text

- ~~**O-X1 — the Unicode version.**~~ — **SETTLED, T-100.** See Q-1.
- ~~**O-X2 — cluster or codepoint in the public width API?**~~ — **SETTLED,
  T-106: `text_width(uint8[])` and `cluster_width(uint8[])`.** The codepoint
  form stays internal, because a caller holding a bare codepoint has already
  lost the variation selector that rules 3 and 4 need.

### Capabilities

- **O-C1 — the initial capability table's contents.** **Open by design:** it is
  *data*, written at cycle 0.4 against real terminals and completed at 0.16,
  not a preference anybody can settle in advance.
- ~~**O-C2 — probe `DECRQM ?1049`?**~~ — **SETTLED, T-107: no.** The alternate
  screen is universal enough that the query buys nothing, and a terminal
  without it ignores both the enter and the leave.

### Input

- ~~**O-I1 — shifted vs unshifted `Char` on the legacy path.**~~ — **SETTLED,
  T-108.** Report `Char('A')` with **no** `SHIFT`: synthesising a modifier the
  terminal did not report is a lie a key binding will trip over. The asymmetry
  between protocols is documented, and `key_matches()` absorbs it at the point
  of use.
- ~~**O-I2 — request kitty bit 4 (alternate keys) by default?**~~ —
  **SETTLED, T-109: no.** The cost is on every keystroke and only a
  keyboard-remapping application needs it; `Caps.kitty_flags` can add it.
- **O-I3 — key repeat on the legacy path.** Indistinguishable from a fast
  press; reported as `Press`. Only the kitty protocol can do better and it
  does. **No action.**

### Screen and style

- **O-R1 — the inline cluster size (14 bytes).** **Open by design:** it is a
  *measurement*, taken at cycle 0.6.0 over a corpus of real terminal content,
  and the number chosen is recorded there with the corpus. What is not
  negotiable, and is not open, is that `Cell` has no owning field.
- **O-R2 — `NTUI_GAP` (8) and `NTUI_EL_MIN` (4).** **Open by design:** chosen
  from sequence lengths and *confirmed by measurement* at cycle 0.8.0, against
  emitted-byte counts on a corpus of realistic frames. Changing either
  re-records every golden, which is the right amount of friction.

### Layout

- ~~**O-L1 — does `Min(k)` grow above its floor?**~~ — **SETTLED, T-110: it
  grows**, like `Fill(1)`, which is what makes `[Min(10), Fixed(3)]` do the
  obvious thing. The alternative reading produces layouts with unexplained
  trailing gaps.

### Runtime

- ~~**O-E1 — may `view` fail?**~~ — **SETTLED, T-111: yes, and the widgets
  substitute rather than fail.** The failure path exists for an application
  that wants it; a bad byte in a label does not abandon the frame.
- **O-E2 — a `Router` for multi-screen applications.** **Open by design:**
  composition is the application's today, through its own screen enum, and a
  helper is added when a consumer asks rather than in anticipation. Cycle 0.15
  is the first consumer that can ask, so it is revisited in 0.15.1's triage.

### Widgets

- ~~**O-W1 — the `Tree` widget's node store.**~~ — **SETTLED, T-101.** See Q-2.
- ~~**O-W2 — image protocols.**~~ — **SETTLED, T-102.** See Q-3.
- **O-W4 — does the log viewer belong inside `nitpick-posix`?** A log viewer
  with follow and search overlaps two POSIX utilities: `more` is the pager,
  `tail -f` is the follow. **Recommendation: its own repository**, because a
  pager exercises the screen model and input and almost no widgets, where a
  viewer with a filter bar and a status line exercises the whole stack — and
  the point of a dogfood consumer is to exercise the whole stack. Decide at
  cycle 0.15's open, before the program exists.
- **O-W3 — accessibility.** A screen reader reads the terminal, not the
  program, so the library's contribution is emitting text in a sensible order
  and keeping the cursor where the focus is — both assertable by the golden
  oracle. Not planned for 1.0; recorded so it is a decision rather than an
  oversight.
