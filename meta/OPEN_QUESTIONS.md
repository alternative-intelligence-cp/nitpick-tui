# Open questions

Everything that is not settled, each with a recommendation, so that nothing
lives only in a conversation. Three prefixes:

| Prefix | Whose |
|---|---|
| `O-x` | **ours** — a design question this project decides, at the cycle named |
| `O-N` | the **compiler's** — a gap in the language or its tooling that `ntui` needs closed, to be raised as a request |
| `Q-` | the **user's** — a question that wants an answer before the work it gates begins |

A question that gets answered moves to `DECISIONS.md` as a numbered decision and
is struck through here with the decision's number, never deleted.

---

## Q — for the user

### Q-1 — the Unicode version to pin
Decided at cycle 0.2 against what is published then; the floor is **15.1.0**,
below which `InCB` does not exist and the Indic conjunct rule GB9c cannot be
implemented. Not really a preference question — the answer is "the latest
stable UCD when 0.2 runs" — but it is recorded because the version becomes a
committed constant that changes rendering when it moves.
**Recommendation:** latest stable at 0.2, recorded in `src/unicode/version.npk`.

### Q-2 — whether `ntui` should ship a `Tree` widget's node store
`WIDGET_MODEL.md` O-W1. Shipping an `arena<T>`-backed store is less work for
the application; taking a trait over the application's own store means a file
browser over a real directory tree does not have to mirror it first.
**Recommendation:** the trait. Decide at cycle 0.12.

### Q-3 — image protocols (Kitty graphics, Sixel, iTerm2 inline)
Out of scope at 1.0 and recorded as wanted. `Caps.sixel` is already carried so
the detection is not re-litigated later. It is a cycle of its own: the three
protocols disagree about placement, scrolling and lifetime, and the cell model
has to learn about a region it does not own.
**Recommendation:** post-1.0, as cycle 1.1, with its own decision batch.

### Q-4 — the release cadence and versioning
`ntui` has no consumers yet, so nothing forces an answer. The compiler restarts
its own version at 0.0 after the switch.
**Recommendation:** `0.x` until the compiler reaches 1.0 and the API has
survived one real application; semantic versioning thereafter, with the
`failsafe` arm list treated as part of the public API — adding an error
identity is a **major** version change, because it breaks every consumer's
compilation.

### Q-5 — whether to build a second library alongside as a consumer
The library's own examples are its first consumer, and they are weak evidence:
an example is written by the person who wrote the API. A real program — a log
viewer, a process monitor, a `git` interface — exercises it honestly.
**Recommendation:** yes, at cycle 0.14, in this repository under `examples/`
rather than as a separate repository, so it moves with the API.

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
- **O-S2 — should `ntui_restore()` also emit `DECSTR` (soft reset)?** For: it
  repairs modes the *application* set that `ntui` does not know about.
  Against: it also repairs modes the user's shell set deliberately before the
  program ran, and the restore's rule is to undo what we changed and nothing
  else. **Recommendation: no.** An application may emit it from its own
  `failsafe` arm after the restore. Decide at cycle 0.3.

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
- **O-B2 — ship as source or as an object.** A `.o` plus a declaration file
  builds faster; source keeps the closed-world link and the whole-program
  verification story. **Settled for now in favour of source**; revisit only if
  build times become a real complaint.

### Text

- **O-X1 — the Unicode version.** See Q-1.
- **O-X2 — should `width()` take a cluster or a codepoint in the public API?**
  **Recommendation:** the public surface is `text_width(uint8[])` over a string
  and `cluster_width(uint8[])` over one cluster; the codepoint form stays
  internal, because a caller holding a bare codepoint has already lost the
  information rules 3 and 4 need. Decide at cycle 0.2.

### Capabilities

- **O-C1 — the initial capability table's contents.** Data, filled at cycle 0.6
  against the real matrix. Not a design question.
- **O-C2 — probe `DECRQM ?1049`?** **Recommendation: no.** The alternate screen
  is universal enough that the query costs more than it buys, and a terminal
  without it ignores both the enter and the leave.

### Input

- **O-I1 — shifted vs unshifted `Char` on the legacy path.** The legacy path
  only ever sees the shifted form (`A` for Shift+a) and cannot know a modifier
  was involved. **Recommendation:** report `Char('A')` with **no** `SHIFT`,
  because synthesising a modifier the terminal did not report is a lie a key
  binding will trip over. Document the asymmetry between protocols and provide
  `key_matches()` as the helper bindings use. Decide at cycle 0.4.
- **O-I2 — request kitty bit 4 (alternate keys) by default?**
  **Recommendation: no.** It adds shifted and base-layout codepoints to every
  event and only a keyboard-remapping application needs them;
  `Caps.kitty_flags` can add it.
- **O-I3 — key repeat on the legacy path.** Indistinguishable from a fast
  press; reported as `Press`. Only the kitty protocol can do better and it
  does. **No action.**

### Screen and style

- **O-R1 — the inline cluster size (14 bytes).** A measurement, not a guess:
  cycle 0.5 measures a corpus of real terminal content and records the number
  it chose. What is not negotiable is that `Cell` has no owning field.
- **O-R2 — `NTUI_GAP` (8) and `NTUI_EL_MIN` (4).** Chosen from sequence
  lengths, to be confirmed by the renderer benchmark at cycle 0.7. Changing
  either re-records every golden, which is the right amount of friction.

### Layout

- **O-L1 — does `Min(k)` grow above its floor?** As specified it does, like
  `Fill(1)`, which is what makes `[Min(10), Fixed(3)]` do the obvious thing.
  The alternative reading produces layouts with unexpected trailing gaps. Both
  readings occur in existing libraries. **Recommendation: it grows.** Decide at
  cycle 0.9.

### Runtime

- **O-E1 — may `view` fail?** As specified it returns `Result<NIL>` like
  everything else. **Recommendation:** it may, and the *widgets* substitute
  rather than fail, so the failure path exists but is not the ordinary one.
- **O-E2 — a `Router` for multi-screen applications.** Composition is currently
  the application's, through its own screen enum. **Recommendation:** defer
  until something written against the library asks for it — the examples being
  the first thing to ask. Revisit at cycle 0.14.

### Widgets

- **O-W1 — the `Tree` widget's node store.** See Q-2.
- **O-W2 — image protocols.** See Q-3.
- **O-W3 — accessibility.** A screen reader reads the terminal, not the
  program, so the library's contribution is emitting text in a sensible order
  and keeping the cursor where the focus is — both assertable by the golden
  oracle. Not planned for 1.0; recorded so it is a decision rather than an
  oversight.
