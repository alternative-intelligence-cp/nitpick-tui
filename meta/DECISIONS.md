# Design decisions

Every settled design decision for `ntui`, with the reasoning, the alternatives
that were considered, and the date. **This is the file to read when something in
the specifications looks unusual**, because it is recorded why.

Referenced as `T-nnn` from the specifications. `D-nnn` in those documents refers
to the **compiler's** `meta/specs/DECISIONS.md`; those are language decisions
and are not ours to amend.

**Rule: a settled decision's text is never rewritten.** A decision that turns
out to be wrong is superseded by a new one that says so and says why; the old
text stays, dated, because it records what was true when it was made. This is
the compiler's D-085/D-202 pattern.

**Numbering is allocation order, grouped by the batch that settled it.**
T-001…T-094 are the founding batch, written with the specification set and
organised below by area. T-100 onward are later batches, each appended whole
with its own heading, because a batch ratified together is a unit and splitting
it across the areas would lose the fact that one conversation settled all of
it. This is the compiler's own shape — its D-194…D-200 and D-201…D-209 are
batches, not areas.

---

## Foundations

### T-001 — the library is `ntui`; the repository is `nitpick-tui`
**2026-09-03.** The module prefix, every public symbol's prefix, and the
eventual package name are `ntui`, matching the ecosystem's `n`-prefix
convention (`nfs`, `nproc`, `nio`, `nvec`, `nsys`, `ncolor`, `ngui`). The
repository keeps the longer name because a repository name is a search term and
`ntui` alone is not one.

*Alternatives:* `tui` (collides with a user's own module of that name and breaks
the convention every other library follows); `nitpick_tui` (verbose at every
import site).

### T-002 — the specifications are the authority
**2026-09-03.** Code that disagrees with `meta/specs/` is a defect in the code.
A specification that is wrong is amended by a decision recorded here, never by
editing the text and moving on. The compiler's own cycle notes record the same
finding repeatedly — the compiler and the thing that describes it have to be
diffed, because reading either alone never reveals the gap — and
`TESTING.md` §3's checks are that diff, applied here.

### T-003 — no terminfo
**2026-09-03.** Capabilities come from a compiled-in table keyed by
`$TERM_PROGRAM` and `$TERM`, corrected at startup by asking the terminal
itself. The system terminfo database is never read.

*Reasoning, in order of weight:* it is a binary format read from files the
program does not control, which is an untrusted-input parser inside a library
whose entire claim is that it has none; its answers depend on machine state,
which would make rendering non-deterministic and the golden suite impossible;
it is routinely wrong about modern terminals, so every library that uses it
also carries a table of exceptions for exactly the capabilities that matter,
which means the table is doing the work; and the sequences that matter are
standardised in practice.

*Given up:* hardware and historical terminals. Stated in `COMPAT.md` §4 rather
than pretended away.

*Alternatives:* parse terminfo (rejected above); hybrid table-then-terminfo
(rejected: two code paths for one question, and the fallback is the one nobody
tests).

### T-004 — immediate-mode rendering with an Elm-shaped loop
**2026-09-03.** Widgets are plain values drawn into a buffer each frame;
application state is the application's own struct; events reach an `update`
that mutates it; `view` draws.

*Reasoning:* **the language has no closures** (D-018), which removes the
retained-tree-plus-callbacks architecture most toolkits use — a callback slot
could hold only a bare function pointer with no captured state. What remains
also happens to be the architecture that is byte-deterministic, testable
without a terminal, and free of the "why did this not update" class of bug that
a dirty-propagating component tree produces.

*Costs accepted:* the whole screen is composed every frame (the renderer's diff
makes that cheap, and an unchanged frame emits zero bytes), and a widget cannot
hold state unless the application holds it for it.

*Alternatives:* a retained component tree over `arena<T>` + `Handle<T>` — which
the language actually supports well — rejected on the closure argument and the
verification argument. Revisitable as a layer *above* the immediate core if
something asks for it.

### T-005 — `harness/` builds and tests `ntui` until `npkg` can
**2026-09-03.** Measured at the compiler's 1.5.0: `npkg build` is the
compiler's own bootstrap ladder with no generic-project path, and
`[dependencies]` is parsed but the loader's dependency-root list is created
empty and never populated, so cross-repository imports do not resolve. A Python
harness drives `npkc`, `llc` and `ld.lld` directly, mirroring
`bootstrap/harness/`'s relationship to `npkg`, and retires the same way — both
running side by side with a parity check before the older is removed.

*Not a dependency violation:* zero-dependency governs the artifact, not the
workbench (the compiler's `ORCHESTRATION.md` §6 says so in as many words).

### T-006 — `ntui` declares its own storage primitives
**2026-09-03.** `Vec<T>`, `Bytes` and `SmallMap<K, V>` live in `src/core/` and
are ours. The compiler's `List<T>` is not imported: it is a compiler internal
whose own header says it exists for the compiler's tables, and reaching into
another project's `src/` couples this library's correctness to a file that is
not a published interface. `Vec<T>` is `List<T>`'s shape, deliberately, because
that shape is right and has been exercised across twenty-two families.

### T-007 — no dependencies, and `[dependencies]` stays empty
**2026-09-03.** The language and its prelude, and nothing else. Including, by
name: not the compiler's `src/`, not the compiler's `lib/` (which is scheduled
to move to an `nlibc` sibling repository, so importing it today is importing a
path that will change), not terminfo, not ncurses. Adding an entry is a
zero-dependency decision and has to be argued as one.

### T-008 — Linux on x86-64 only, at 1.0
**2026-09-03.** The device layer is Linux syscalls with Linux ioctl numbers,
`signalfd4`, `eventfd2` and the kernel's 36-byte `termios`. A BSD port is a
different structure, different ioctl numbers and no `signalfd`; it is a cycle
with its own decision rather than something left half-done behind a
conditional. `aarch64` Linux is the first port to consider — the syscall
numbers differ, the structures do not.

---

## The device

### T-010 — `/dev/tty` first, with an explicit inherited fallback
**2026-09-03.** `ntui` opens `/dev/tty` with `O_RDWR | O_NOCTTY | O_NONBLOCK |
O_CLOEXEC`. It is a new open-file description, so setting `O_NONBLOCK` on it
affects nobody else — where setting it on an inherited descriptor rides the
shared description into the parent's shell, which is exactly why the language
makes descriptors 0/1/2 the one stated exception to D-071 and leaves them
blocking. It also survives redirection, so a TUI can still be scripted.

When `/dev/tty` cannot be opened, `ntui` falls back to a `F_DUPFD_CLOEXEC`
duplicate of the first of 2, 1, 0 that is a terminal, **without** setting
`O_NONBLOCK`, and records `TtySource.Inherited` in the value. The consequence —
reads gated on readiness first, writes able to block the thread — is a property
of the value rather than a hidden mode.

*Alternatives:* refuse without `/dev/tty` (rejected: `ssh -T`, some containers
and some `sudo` configurations genuinely have a tty fd and no controlling
terminal); a reader thread feeding a channel (rejected: a second code path and a
thread for a case the reactor already handles).

### T-011 — signals through `signalfd`, and no handler is ever installed
**2026-09-03.** `SIGWINCH`, `SIGTERM`, `SIGHUP`, `SIGINT`, `SIGQUIT`,
`SIGTSTP` and `SIGCONT` are blocked with `rt_sigprocmask` and read as
128-byte records from a `signalfd4` descriptor that lives in the reactor beside
the terminal. **`SIGPIPE` is in the set too — see T-113**, which corrects this
entry's original claim that it could safely be left out.

*Reasoning:* the language has no signal-handling facility at all, and adding
one would mean async-signal-safety questions inside a runtime that allocates.
`signalfd` removes the question rather than answering it, and puts signal
delivery in the same wait as everything else.

*Known consequence:* the blocked mask is inherited across `execve`, and the
floor's `clone_exec` has no slot for a signal mask, so a spawned child would
inherit it. `ntui_suspend()` unblocks around a spawn and `ntui_resume()`
re-blocks and re-queries the size. Recorded as O-N1.

### T-012 — the restore record
**2026-09-03.** Terminal state lives in one statically allocated, fixed-size
record: an atomic liveness word, the raw descriptor, the captured 36-byte
original `termios`, a bitmask of the modes turned on, and the original autowrap
state. It allocates nothing and is written before the first mode change.

*Reasoning:* `defer` does not run on a trap (D-014) and a trap is a
whole-program event in which no task resumes (D-063), so the only code
guaranteed to run is `failsafe`, and `failsafe` runs in a process whose heap may
be corrupt. A restore that allocates, locks or awaits cannot be called from
there. This is the same mechanism the language already uses for the allocation
registry (D-151) and the driver registry (D-188), applied to the one piece of
global state a terminal program owns.

### T-013 — `ntui_restore()` is idempotent and has three callers
**2026-09-03.** It is called by the `Terminal`'s scope-exit drop (the normal
path), by `ntui_suspend()`, and — required of every consuming application, as
the **first** statement — by `failsafe`. Idempotence is an atomic
compare-exchange on the record's liveness word, which is what lets all three
exist without coordinating.

*Why require it of the application at all:* the compiler cannot enforce it. The
requirement is therefore stated in exactly one sentence, demonstrated in every
example, and checked by a conformance test — and repeated nowhere else, so that
the one sentence is the thing people learn.

### T-014 — raw mode edits the captured original, and restores it exactly
**2026-09-03.** The termios written is the captured original with a stated set
of bits cleared and set (`TERMINAL_MODEL.md` §3, which lists every one and why),
never a constructed "sane" state. A terminal may arrive with an unusual speed,
line discipline or control characters, and inventing a replacement discards
information the program was handed.

### T-015 — `TIOCGWINSZ` is the size, and there is no default
**2026-09-03.** The ladder is: the ioctl; then `$LINES`/`$COLUMNS`; then the
in-band `CSI 18 t` query; then `ETuiSize`. **No 80×24 fallback.** A frame drawn
at a guessed size onto a terminal of another size is worse than a refusal, and
the application is in a far better position to decide whether to proceed.

### T-016 — suspend and resume are first-class
**2026-09-03.** `ntui_suspend()` / `ntui_resume()` are public and are what an
application calls around a spawned editor, a pager, a shell escape, or its own
Ctrl-Z. An externally delivered `SIGTSTP` drives the same pair through the
signal fd. Both paths meet in one function.

*Reasoning:* with `ISIG` cleared, Ctrl-Z is byte `0x1A` and not a signal, so
the keyboard path and the `kill -TSTP` path are genuinely different arrivals for
one behaviour; giving them two implementations is how one of them rots.

### T-017 — `TCSETSW` and `TCSETSF` are never used
**2026-09-03.** Both wait for pending output to drain, and that wait is
unbounded and not interruptible by the deadline machinery — unacceptable on the
trap path and undesirable everywhere else. `TCSETS` applies immediately;
`TCFLSH` with `TCIFLUSH` discards stale input on its own without touching
output.

### T-018 — the error budget is nine
**2026-09-03.** `ETuiDevice`, `ETuiNotATerminal`, `ETuiSize`, `ETuiInput`,
`ETuiOutput`, `ETuiEncoding`, `ETuiGeometry`, `ETuiCapacity`, `ETuiState`.

*Reasoning:* REACH-002 makes every `error:` that can reach `failsafe` a named
arm every consuming program's `failsafe` must carry, in a language where
forgetting one is a compile error. A library that declares thirty errors makes
thirty arms mandatory in every program that imports it. Nine is a ceiling, not
a target; a tenth requires a decision saying why the failure is one a shutdown
handler would treat differently from all nine. Distinctions that are for the
*caller* ride as detail fields on the returned value.

*Corollary:* module decomposition is part of the budget, because REACH is
import-scoped — `ntui/text.npk` and `ntui/layout.npk` declare no errors at all,
so a program importing them for measurement owes no arms.

---

## Text

### T-020 — the unit of the screen is the grapheme cluster
**2026-09-03.** Not the byte, not the codepoint. A cell holds one cluster and
every API that talks about a screen position talks in cells.

### T-021 — Unicode tables are generated and committed as Nitpick source
**2026-09-03.** `tools/gen_unicode.py` reads a pinned UCD and writes
`src/unicode/*.npk`, which are committed. A build needs the compiler and
nothing else: no Python, no network, no `/usr/share/unicode`. The generator is
checked rather than trusted — the harness re-runs it and requires byte
identity, the same instrument the compiler uses for its builtin signature
table. The Unicode version lives in exactly one file and upgrading it is a
recorded decision with a re-run golden suite.

### T-022 — the width algorithm, stated
**2026-09-03.** Controls refused; VS16 promotes to 2 and VS15 demotes to 1;
`Emoji_Presentation=Yes` is 2; East Asian `W`/`F` is 2; `Mn`/`Me`/`Cf` and
`Default_Ignorable` are 0; everything else is 1. `Ambiguous` is 1 by default
and 2 under a capability, because it is right for Latin locales and wrong in a
CJK locale on a terminal configured for it — and being wrong there tears every
box in the program.

### T-023 — invalid UTF-8 becomes U+FFFD by the maximal-subpart rule
**2026-09-03.** One replacement per maximal subpart, which is the Unicode
recommended practice and what browsers do. Bytes are never dropped silently and
never trusted: a terminal is an untrusted input device and a paste can carry
anything.

### T-024 — ZWJ width is measured, not believed
**2026-09-03.** An emoji ZWJ sequence renders as one double-width glyph on a
composing terminal and as several on one that does not, and **no static table
can answer it** — the answer is a property of the font and the terminal. The
static rule is 2; the calibration writes four probe strings at startup and asks
`CSI 6 n` where the cursor landed, recording the terminal's actual behaviour
for a ZWJ family, a regional-indicator pair, a VS16 base, and a wide CJK
character; an application may override.

*Cost:* one round trip at startup, bounded by 100 ms, skipped on the headless
device. Accepted because a wrong width is a visible defect and 100 ms is not.

### T-025 — no normalisation, no case mapping beyond ASCII, no collation, no bidi
**2026-09-03.** Normalising a user's text behind their back changes their data;
case mapping and collation are locale questions and this library has no locale;
right-to-left reordering is owned by the terminal on most emulators and the two
doing it at once is worse than either. `ntui` renders logical order. Each is
stated as absent rather than attempted badly, and revisiting any of them is its
own decision.

---

## Input

### T-030 — the decoder is a pure incremental function
**2026-09-03.** `(state, bytes, now_ns) → (events, state)`. No I/O, no
descriptor, no clock of its own — the timestamp is supplied by the caller, so
the escape timeout is a function of a value a test controls. Everything about
the input suite follows: fixtures replay deterministically, timing included,
and a fuzzer drives the decoder directly.

### T-031 — the escape ambiguity is resolved by an explicit deadline
**2026-09-03.** A lone `ESC` with no continuation within `NTUI_ESC_TIMEOUT`
(25 ms, application-overridable) is the Escape key. Not a heuristic, not a
byte-count guess. Over a slow link the default is too short and the application
raises it; under the kitty protocol the ambiguity does not exist at all
(Escape arrives as `CSI 27 u`) and the timeout is set to zero.

### T-032 — the kitty keyboard protocol is preferred; flags are pushed, not set
**2026-09-03.** It is the only encoding that reports key releases,
distinguishes every modifier combination, and carries the text a press
produced. `ntui` pushes flags `1 | 2 | 8 | 16` = **27** (disambiguate, event types,
all keys as escape codes, associated text) with `CSI > 27 u` and pops with
`CSI < 1 u`, so an application running inside another that had its own flags
set does not clobber them. Bit 4 (alternate keys) is not pushed by default.

*The pairing that matters:* requesting "all keys as escape codes" without
"associated text" produces an application that cannot type, because the
terminal stops sending plain bytes. This is the most common way to get the
protocol wrong.

### T-033 — SGR mouse only is requested; the legacy encodings are decoded
**2026-09-03.** `ntui` requests 1006 (and 1016 where available) and nothing
else. X10, 1005 and urxvt 1015 are decoded, because a previous program may have
left the terminal in one of them, and never requested — X10's coordinates break
above column 223, which is the whole reason.

### T-034 — everything the terminal answers is an event
**2026-09-03.** DA, DSR, DECRQM, XTVERSION, kitty flags, text-area reports and
OSC replies are `Reply` events on the ordinary stream. The capability probe
consumes what it asked for by reading events until DA1 and **pushes everything
else back in order** — so a user who typed during startup keeps their
keystrokes. A library that eats the first keypress is one everybody notices.

### T-035 — a paste is one event
**2026-09-03.** Between `CSI 200 ~` and `CSI 201 ~` nothing is interpreted, no
key events are produced, and the content accumulates in the decoder's paste
buffer with the event carrying an offset and a length into it. Bounded at 1 MiB
by default; exceeding it truncates and flags rather than failing, because a
user pasting a large file into a text field should get a usable prefix and a
signal. Content is not validated as UTF-8 by the decoder — a paste can be
anything.

### T-036 — `0x7F` is Backspace and `0x0D` is Enter
**2026-09-03.** `0x08` is Backspace with Ctrl; `0x0A` is Ctrl-J. With `ICRNL`
cleared the Enter key sends `0x0D` and nothing else does, so the mapping is
exact rather than a guess, and Ctrl-H stays distinguishable — which the
alternative reading does not allow.

### T-037 — mouse coordinates are 0-based cells at the decoder
**2026-09-03.** The wire is 1-based; the conversion happens once and every
layer above works in the same space as the layout engine and the cell buffer. A
library that leaks 1-based wire coordinates produces exactly one off-by-one per
application.

---

## Style and screen

### T-040 — `Color` has four kinds, and `Ansi` is not `Indexed`
**2026-09-03.** `Default`, `Ansi(0..15)`, `Indexed(0..255)`, `Rgb(r,g,b)`.
`Ansi(1)` emits `SGR 31`, which the terminal renders with its *configured* red
and honours the bold-is-bright convention; `Indexed(1)` emits `SGR 38;5;1`,
which is a different request. Folding them would make the first inexpressible.
`Default` is neither black nor white — it is the terminal's own, which is what
keeps a themed or transparent background working.

### T-041 — attributes are a plain `uint16` with named constants
**2026-09-03.** A library **cannot** declare a flag family: D-230 makes
`TY_FLAGS` four compiler-known families whose members are generated prelude
constants. `Mods` and `attrs` are therefore plain integers with named constants
and predicates, and do not pretend otherwise.

### T-042 — reset-and-reapply whenever an attribute must be removed
**2026-09-03.** The renderer emits only additions when the target's attribute
set is a superset of the pen's and neither colour becomes `Default`; otherwise
it emits `CSI 0 m` and the target in full.

*Reasoning:* `SGR 22` turns off **both** bold and dim, so there is no sequence
that removes bold while leaving dim — a renderer emitting individual off-codes
cannot express every transition. The rule is deterministic, has no branch a
reader must reason about, and costs a few bytes on transitions that remove an
attribute, which is not the common case.

### T-050 — `Cell` is a 32-byte plain value with 14 inline bytes and a spill
**2026-09-03.** Fourteen holds every cluster that occurs in practice; beyond
that `len == 255` and the first four bytes index the buffer's cluster pool.

*The non-negotiable half:* `Cell` has no owning field. A `string` there would
make the cell move-only (TYPE-046) and a grid of move-only cells cannot be
copied, cleared or diffed. The same rule governs `Style` and `Event`.

### T-051 — wide characters occupy two explicit cells
**2026-09-03.** A width-2 cluster sets `CELL_WIDE_LEAD` and overwrites the next
cell with a `CELL_WIDE_TAIL` blank of the same style. Writing over half a pair
blanks the other half. Nothing infers width at render time; the question is
answered once, when the text is written.

### T-052 — the renderer is a pure function
**2026-09-03.** `(front, back, caps, cursor) → bytes`. No environment, no
syscall, no clock, no read from the terminal. This is what makes the golden
suite possible and it is the constraint every other rule in
`SCREEN_MODEL.md` exists to protect.

### T-053 — damage is one span per row
**2026-09-03.** Not per cell (a bitmap costs a branch per write and the
renderer would coalesce anyway) and not per screen (which makes every frame a
full repaint). A write marks damage even when the value is unchanged; the diff
decides what actually changed. Over-approximating is the safe direction.

### T-054 — synchronized output wraps every frame, unconditionally
**2026-09-03.** One place, no per-widget or per-region use, no size condition.
A terminal that supports DEC 2026 renders the frame atomically; one that does
not ignores both sequences. A frame that resolves to no changes emits nothing
at all, wrapper included.

### T-055 — no scroll optimisation at 1.0
**2026-09-03.** `CSI S`/`T` and `CSI L`/`M` are a large win for a scrolling
list and the single most common source of rendering bugs in mature TUI
libraries: they require modelling the terminal's scrollback, margins and
clearing behaviour, all of which vary, and they turn a pure diff into a
stateful transformation of the front buffer. Deferred to its own cycle, gated
on the golden oracle being able to prove a scroll produced the same screen as a
repaint.

### T-056 — a `Surface` is built by struct literal and never returned
**2026-09-03.** `Surface` holds a `Buffer->` borrow, and a function returning
one launders a borrow upward, which the escape analysis refuses (D-004).
Sub-surfaces are `Surface{ buf: s.buf, area: child }` at the call site. This
shapes the whole drawing API and is verified by a probe program in cycle 0.0
before anything is built on it.

---

## Layout

### T-060 — geometry is `uint16` and every rectangle operation is total
**2026-09-03.** A coordinate cannot be negative and cannot be spelled negative;
`x + w` computes in `uint32` and clamps. An empty rect is legal everywhere and
no function fails because one is empty — a window too small for a layout is an
everyday condition, not an error. A margin larger than its rect gives `w = 0`,
never an inverted rect.

### T-061 — the layout solver is integral and its remainder rule is stated
**2026-09-03.** No floating point: percentages and ratios are integer divisions
of the available extent, surplus is distributed among fillers proportionally
with the remainder going **one cell each to the first segments in declaration
order**, and a deficit shrinks in **reverse declaration order** down to each
segment's floor.

*Alternatives:* a Cassowary simplex over floats, which is what several widely
used libraries do — rejected because results depend on iteration order and can
differ by a cell between builds, and a layout that differs by a cell between
builds is a golden test that cannot exist. Largest-remainder distribution —
rejected as a tie-break away from non-determinism, where declaration order is
predictable by a user.

---

## The runtime

### T-070 — `update` is `async`, `view` is not
**2026-09-03.** An application waiting on a file, a socket or a subprocess does
it inside `update` with an ordinary `await` and needs no command indirection.
`view` may not: drawing is a pure function of state, and a `view` that could
suspend could see a half-updated model.

### T-071 — `Ctx` is a closed set of runtime requests
**2026-09-03.** Quit, redraw, cursor position and style, title, mouse mode,
timers, suspend, bell. Application work does not go through it. The closed set
is what makes commands expressible without closures — an open `Cmd` enum the
application extends would require the runtime to interpret types it cannot know
— and adding an entry is a decision because each one is a thing the runtime
must know how to do.

### T-072 — background work reaches the loop through an eventfd and a bounded queue
**2026-09-03.** The runtime owns an `eventfd2(0, EFD_CLOEXEC | EFD_NONBLOCK)`
and watches it in the same `suspend_io` as the terminal and the signal fd;
`ntui_post` appends to a mutex-guarded fixed-capacity queue and writes eight
bytes. An eventfd rather than a channel because the loop's wait is over
descriptors and there is no `select` (D-072). A full queue returns
`ETuiCapacity` — it does not block the poster and does not drop silently,
because a background task that can outrun the UI has a design question to
answer and the library's job is to make it visible.

### T-073 — `always_redraw` defaults to on
**2026-09-03.** Every event batch ends in a redraw unless the application turns
it off. `ctx.redraw()` is a thing an application forgets, and the failure — a
screen that does not update — is far more confusing than a wasted diff that
emits zero bytes.

### T-074 — the order of events within one wake is fixed
**2026-09-03.** Signals, then terminal input in byte order, then posted
messages in post order, then expired timers by deadline and then id. A resize
must be seen before the events that follow it are drawn, and a fixed order is
what makes the whole loop replayable.

---

## Testing

### T-090 — there is a headless device
**2026-09-03.** `ntui_headless(rows, cols, caps)` returns a `Terminal` whose
output is a byte sink and whose input is a byte source the test supplies.
Everything above `src/term/` is exercised with no terminal at all. This is the
reason the device is a module boundary rather than calls scattered through the
library, and it is what makes the suite runnable in CI, under a debugger, and
forty times over.

### T-091 — the golden oracle is a round trip through a mini-VT
**2026-09-03.** A test builds a buffer, renders it, compares the bytes against
a committed golden, **parses those bytes with a miniature VT written in
Nitpick**, and compares the resulting screen against the buffer it started
from. A renderer bug that produces plausible bytes — a cursor position off by
one, a missing style reset, an emitted wide-character tail, an SGR that turned
off dim along with bold — passes byte equality the day the golden was recorded
and fails the round trip immediately.

The oracle **fails loudly on any sequence it does not recognise**, so a
renderer that starts emitting something new has to teach it — which is the
friction that keeps the emitted-sequence table honest.

### T-092 — expectations live in the test file and assert on codes
**2026-09-03.** The compiler's marker grammar, marker for marker, so a reader
moving between the repositories reads one thing. Never assert on message text.
The set of codes a rejection test reports must **equal** the set its
expectations name (D-237's rule): the compiler ran the weaker subset rule for
six cycles and found, when it tightened, that 17 of 131 files were reporting
something nobody had asserted.

### T-093 — generated tables are checked by regeneration
**2026-09-03.** The harness re-runs every generator and requires the committed
output to be byte-identical. A hand-edited generated file is the failure this
prevents.

### T-094 — a compatibility claim requires a fixture directory
**2026-09-03.** A terminal appears in `COMPAT.md`'s matrix only when
`tests/fixtures/input/<terminal>/` exists, its cases pass, and its capability
row was written from a measured probe transcript rather than from
documentation. A claim nobody can check is a claim that goes stale silently.

---

# The second batch — ratified 2026-09-03

Thirteen decisions settled in one pass, the day after the founding batch, when
the recommendations standing in `OPEN_QUESTIONS.md` were accepted as written.
Five of them (T-100 … T-104) were questions put to the project's author; the
remaining eight were design calls this project owns and had carried a
recommendation since the specification set was written.

**What was deliberately NOT settled here**, and why, because a batch that
settles everything is a batch that guessed: the inline cluster size (O-R1) and
the renderer's two constants (O-R2) are **measurements**, taken in the cycles
that can take them; the capability table's contents (O-C1) are **data**,
written against real terminals; the migration off the harness (O-B1) is
**gated** on the compiler's tooling; a multi-screen `Router` (O-E2) waits for a
consumer to ask; and O-N1 … O-N4 are the **compiler's**, not ours to decide.

### T-100 — the Unicode version is the latest stable UCD at cycle 0.2, with 15.1.0 as the floor
**2026-09-03, settling Q-1 / O-X1.** The version is recorded in exactly one
file, `src/unicode/version.npk`, as a `pub fixed string:UNICODE_VERSION`, and
appears in the header comment of every generated table.

**15.1.0 is a hard floor**: below it the `InCB` property does not exist and
rule GB9c — the Indic conjunct break — cannot be implemented at all, so a
lower version is not a weaker guarantee but a missing one.

**Upgrading is a decision with a regenerated table set and a re-run golden
suite**, because a width that changes is a rendering that changes and every
committed golden encodes the old answer. The cycle that upgrades records the
diff in cell counts, so an upgrade that moves nothing is visible as such.

### T-101 — `Tree` takes a model trait; it does not ship a node store
**2026-09-03, settling Q-2 / O-W1.** The widget declares a trait the
application implements over its own data — children of a node, a node's label,
whether it has children — rather than an `arena<T>`-backed store the
application must mirror into.

*Reasoning:* the motivating case is a file browser over a real directory tree,
and a shipped store would make that program maintain two representations of the
same hierarchy and keep them in sync. The language's arenas are excellent for
tree data and remain available to an application that wants one; that is a
choice the application gets to make.

*Cost accepted:* a trait per model is more ceremony for the trivial case (a
static menu), and the library's examples carry a small in-memory implementation
to make that case one line.

### T-102 — image protocols are post-1.0, as cycle 1.1, with their own batch
**2026-09-03, settling Q-3 / O-W2.** Kitty graphics, Sixel and the iTerm2
inline protocol are out of scope at 1.0 and are not deferred by silence: they
are cycle 1.1 in the post-1.0 map, opening with a decision batch of their own.

*Reasoning:* the three protocols disagree about placement, scrolling behaviour
and image lifetime, and supporting them means the cell model has to learn about
a screen region it does not own — which touches the renderer's diff, the damage
model and the scroll path all at once. That is a cycle, not a widget.

`Caps.sixel` is carried from cycle 0.4 so the *detection* is not re-litigated
when the cycle opens.

### T-103 — versioning, and adding an error identity is a MAJOR version
**2026-09-03, settling Q-4.**

- `0.x` until the compiler reaches its own 1.0 **and** the API has survived one
  real application (cycle 0.15's dogfood program). Before that, anything may
  move.
- Semantic versioning thereafter.
- **The `failsafe` arm list is part of the public API.** Adding a public
  `error:` identity is a **major** version change, because REACH-002 makes it a
  new mandatory `pick` arm in every consuming program's `failsafe` — a source
  break in every consumer, enforced by the compiler.

That last rule is the one thing about `ntui` a consumer most needs to know, and
no library in any other language has to state it. It is published in the 1.0
contract (`meta/roadmap/1.0/README.md` §1.0.1) and it is the practical teeth
behind T-018's budget of nine: the budget is not a style guide, it is the thing
that keeps the major version from moving.

### T-104 — the dogfood application is cycle 0.15, in this repository
**2026-09-03, settling Q-5.** A real program — a log viewer with search,
filtering and follow — written against the library as a consumer, in
`examples/`, before 1.0.

*Reasoning:* an example written by the person who wrote the API is weak
evidence; it demonstrates the features the author was thinking about. A program
with a purpose finds what is missing, what is awkward, and what is wrong.

*In this repository, not a separate one*, so it moves with the API and a
breaking change breaks it in the same commit rather than six months later.
Every friction it meets is recorded and triaged as a defect, a gap, or an
accepted cost — and an accepted cost that nobody warned about is a defect in
the documentation.

### T-105 — `ntui_restore()` does not emit `DECSTR`
**2026-09-03, settling O-S2.** The restore undoes what `ntui` changed —
the captured `termios` and the modes in its own bitmask — and nothing else. It
does not issue a soft reset.

*Reasoning:* a soft reset would also repair modes the **user's shell** set
deliberately before the program ran, which is the restore's rule (S-2)
inverted. An application that wants one emits it from its own `failsafe` arm,
after the restore, where it is that application's decision and is visible as
such.

### T-106 — the public width API is `text_width` and `cluster_width`
**2026-09-03, settling O-X2.** `text_width(uint8[]) → uint32` over a string and
`cluster_width(uint8[]) → uint8` over one cluster. The codepoint form stays
internal to `src/unicode/`.

*Reasoning:* rules 3 and 4 of the width algorithm (T-022) need the variation
selector, which is a *later* codepoint in the cluster. A caller holding a bare
codepoint has already lost the information the algorithm needs, so exposing
that form would be exposing a function that is wrong for emoji and right for
nothing the library measures.

### T-107 — no DECRQM probe for `?1049`
**2026-09-03, settling O-C2.** The alternate screen is not queried. It is
universal enough that a query costs a parameter in the probe and buys nothing,
and a terminal without it ignores both the enter and the leave — which is a
degraded but coherent result, not a corrupted one.

### T-108 — the legacy path does not synthesise `SHIFT`
**2026-09-03, settling O-I1.** On the legacy encodings a shifted letter arrives
as `Char('A')` with **no** `MOD_SHIFT`, because the terminal reported no
modifier and inventing one is a lie a key binding will trip over. The kitty
protocol reports the unshifted key plus the modifier and is decoded as it
arrives.

The asymmetry between protocols is **documented rather than papered over in the
data**, and `key_matches()` is the helper that papers over it *at the point of
use* — a binding asks "is this Shift+A" and the helper answers correctly under
either encoding. One place absorbs the difference, and it is the place that
knows what the question means.

### T-109 — kitty bit 4 is not pushed by default
**2026-09-03, settling O-I2.** *Report alternate keys* adds the shifted and
base-layout codepoints to every event. Only a keyboard-remapping application
needs them, the cost is on every keystroke, and `Caps.kitty_flags` lets such an
application add it. The default stays `1 | 2 | 8 | 16` = 27 (T-032).

### T-110 — `Min(k)` grows above its floor, like `Fill(1)`
**2026-09-03, settling O-L1.** A `Min(k)` segment takes at least `k` and then
participates in the surplus distribution with weight 1.

*Reasoning:* it is what makes `[Min(10), Fixed(3)]` do the obvious thing — the
first pane takes everything the second does not. The alternative reading —
`Min` takes exactly its floor unless something forces it wider — produces
layouts with unexplained trailing gaps, and both readings occur in existing
libraries, so the choice had to be made and written down rather than
discovered.

### T-111 — `view` may fail, and widgets substitute rather than fail
**2026-09-03, settling O-E1.** `view` returns `Result<NIL>` like every other
function, so the failure path exists and an application can use it. But the
**widgets** do not take it: malformed text renders as U+FFFD, a control
character renders as its control picture, an over-large cluster becomes a
blank. A frame is not abandoned because one label was wrong.

*Reasoning:* the two halves answer different questions. "Can drawing fail?" —
yes, and pretending otherwise would force a widget to trap where it could
report. "Should a bad byte in a label stop the frame?" — no, because the frame
is how the user finds out something is wrong.

### T-112 — `ntui` ships as source
**2026-09-03, settling O-B2.** Not as an object plus a declaration file.

*Reasoning:* source keeps the closed-world link intact (D-011's
undefined-symbol scan sees every symbol the program contains) and keeps
whole-program verification available, which is the point of the language. An
object would build faster and would put a binary between the consumer and the
evidence.

*Revisit only if build times become a real complaint*, from someone building
a real program — not on principle.

---

# The third batch — corrections, 2026-09-03

### T-113 — `SIGPIPE` is blocked, correcting T-011's original claim
**2026-09-03.** T-011's entry, and `TERMINAL_MODEL.md` §5's original D-15, both
stated that `SIGPIPE` was deliberately left unblocked *"because the floor
already returns `EPIPE` in `Result.err`."* **That rationale is false.** The
error was found by the `nitpick-sockets` planning pass, which measured it
instead of accepting it.

*The measurement:* `runtime/npkrt.ll` contains **no `rt_sigaction` call
anywhere**. The runtime installs no signal disposition for anything, so
`SIGPIPE`'s default is live and its default is to terminate the process.
`lib/nbridge.npk` states the consequence in its own source — it passes
`MSG_NOSIGNAL` on every send precisely because *"a plain write to a socketpair
whose peer died raises SIGPIPE, which terminates the runtime by default"* — and
records that *"the first EPIPE schedule proved it by taking the whole process
down."*

*Why it did not show up in this library's own reasoning:* the primary path is
`/dev/tty`, and a write to a hung-up tty returns `EIO`, not `SIGPIPE`. The
claim was true of the case being thought about and false in general, which is
the most durable kind of specification error — and the reason this project
diffs its documents against the thing they describe rather than re-reading
them.

*Where it actually bites:* the **inherited** path (`TERMINAL_MODEL.md` D-3),
where the descriptor is a dup of 2, 1 or 0 and may be a pipe or a socket.
`myapp | head` is the everyday case: `head` exits, the next frame's write
raises `SIGPIPE`, and the program dies **without restoring the terminal** —
which is the one failure this library's entire safety story exists to prevent.
`MSG_NOSIGNAL` does not help, being a flag on `send(2)` where this library uses
`write(2)`.

*The decision:* `SIGPIPE` joins the blocked set and the `signalfd` watch, and
its record is **drained and discarded** — it produces no `Event`, because the
error a caller acts on is the `EPIPE` the write already returned. Blocking is
what makes that return happen rather than what reports it.

*Alternative declined:* leaving it unblocked and documenting the hazard. A
library that takes over the terminal and then dies on a broken pipe without
restoring it has failed at the one thing it promised, and "documented" is not a
mitigation for that.

*Note on scope, because the sibling library reaches the opposite conclusion and
both are right:* `nitpick-sockets`' `SK-013` passes `MSG_NOSIGNAL` on every
send and explicitly declines to change process-wide signal state. That library
is a passive one that must not alter its host's disposition; this one already
blocks seven signals, owns the terminal, and restores the mask at teardown and
around every suspend. The transferable rule is not "block `SIGPIPE`" — it is
**know what the default disposition is**, and neither library did until it was
measured.
