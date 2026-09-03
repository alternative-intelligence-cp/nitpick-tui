# Building, testing, and the module conventions

How `ntui` is built today, how it will be built when the tooling catches up,
and the file-and-import conventions everything in `src/` follows.

---

## 1. What cannot build this yet, measured

Read at the compiler's commit for cycle 1.5.0 (2026-09-03):

- **`npkg build` is the compiler's own bootstrap ladder.** It assembles
  `runtime/npkrt.ll` and `bootstrap/seed/stage1.ll` into a builder, has that
  builder compile `[build] entry`, scans, links, and names the result `npkc`.
  There is no generic-project path and no `target = "library"` behaviour;
  the key is accepted by the schema and read by nothing.
- **`[dependencies]` resolves to nothing.** The loader's dependency-root list
  (`RootList`, `src/frontend/resolve_path.npk`) is created empty in
  `src/driver/pipeline.npk` and `rootlist_add` is never called from anywhere.
  A `use "ntui/term.npk"` path — the dependency-root form — therefore resolves
  against an empty set. Only `./` and `../` paths work.
- **`npkg` has no `install` and no `update`.** `npkg update` is the only
  command BUILD_REFERENCE gives version resolution to, and it does not exist.

None of this is a criticism of `npkg`; it was built at 1.4.8 to run the
compiler's own suite, and D-206 scoped it to exactly that. It is written down
because a plan that assumes tooling it has not checked is a plan that discovers
the gap at the first step.

**Decision T-005: `harness/` builds and tests `ntui` until `npkg` can, and
retires into it.** That is precisely the relationship `bootstrap/harness/` has
to `npkg` in the compiler repository, including the part where both run side by
side and a parity stage holds them to each other before the older one is
retired. Writing the harness in Python is not a dependency violation:
**zero-dependency governs the artifact, not the workbench** (the compiler's
`ORCHESTRATION.md` §6 says so in as many words, about valgrind), and the
compiler's own harness is Python for the same reason.

---

## 2. The build, step by step

```
src/lib.npk  (and every module it reaches by `use`)
   → npkc              →  build/ntui.ll          the emitted LLVM IR text
   → opt -O2           →  build/ntui.opt.ll      only on the check leg
   → llc               →  build/ntui.o           at the manifest's flags
   → undefined-symbol scan against the runtime allowlist
   → ld.lld -static    →  build/<program>        one program object + npkrt.o
```

**Rule B-1.** Every tool invocation is built from `nitpick.toml`'s
`[toolchain]` lists. No tool ever runs at its own defaults — `llc` defaults to
`-O2` and would optimise a build the manifest declined, which cost the compiler
project a measured 25× on one module.

**Rule B-2.** The undefined-symbol scan is a **build step, not a test**. Every
object is scanned and the build fails on any undefined symbol outside the
runtime allowlist derived from `runtime/npkrt.ll`'s own `define`s plus `main`.
This is what makes "no C, ever" a structural property of `ntui` rather than a
convention, and it is the check that would catch a `sys` wrapper accidentally
lowered to a libcall by `opt`.

**Rule B-3.** The optimised leg runs on every program, every time: the same
program re-emitted through `opt -O2` + `llc -O2` must produce the **same exit
code**, and the zero-dependency scan is repeated on the optimised object
because `opt` may mint libcalls. This is the compiler's 1.3.8 instrument, and
its first run there found a real defect that had passed for six cycles.

**Rule B-4 — reproducibility.** Two builds of the same tree from different
working directories produce byte-identical IR. `ntui` inherits this from the
compiler (D-078, D-204, D-236) and the harness has a `repro` stage that
measures it, because a property nobody measures is a property nobody has.

---

## 3. Test stages

The harness mirrors the compiler's stage vocabulary
(`BUILD_REFERENCE.md` §7.1) so that the eventual move to `npkg` is a change of
runner and not a change of suite.

| Stage | Directory | Passes when |
|---|---|---|
| `parse` | every `.npk` in the tree | accepted by `tools/parse_check` with no diagnostic |
| `accept` | `tests/conformance/` | accepted by `tools/check` in silence |
| `check` | `tests/rejection/` | refused by the frontend with **exactly** the expected codes |
| `program` | `tests/unit/` | emitted, scanned, assembled, linked, run at -O0 and again under `opt -O2`, the same exit both times |
| `golden` | `tests/golden/` | as `program`, and the rendered byte stream matches the committed golden file, and the mini-VT's screen matches the committed screen |
| `fixture` | `tests/fixtures/` | built and never run; its uppercased stem becomes an `// argv:` token |
| `device` | `tests/unit/device/` | as `program`, but **skipped with a loud line when no terminal is available**, never silently |

**Rule B-5 — expectations live in the test file.** The marker grammar is the
compiler's, marker for marker, so a reader moving between the repositories
reads one thing:

```
// expect-exit: 7            the exit a run must produce (0 when absent)
// expect-error: NITPICK-TYPE-046
// expect-error-at: 14:9
// expect-note: …
// stress: 40                run it that many times, the SAME answer every time
// argv: …
// expect-golden: name       the golden pair this test asserts against
```

**Rule B-6 — assert on codes and exit codes, never on message text.** Messages
must stay free to improve.

**Rule B-7 — unexpected diagnostics fail a test as surely as missing ones**
(D-237). The set of codes a rejection test reports must **equal** the set its
expectations name. The compiler ran the weaker subset rule for six cycles and
found, when it tightened, that 17 of 131 rejection files were reporting
something nobody had asserted.

**Rule B-8 — the harness is itself tested.** A self-check feeds it wrong
expectations — wrong code, wrong line, wrong exit, a negative test that
compiles, a golden that does not match — and requires it to report every one as
a failure. A suite that only ever agrees with what it is handed reports green
while checking nothing.

**Rule B-9 — a concurrency or timing test runs 40 times, not once.**
`// stress: N`. Two of the compiler's most serious defects hid behind single
green runs and neither reproduced in fewer than about twenty.

---

## 4. Dependencies

**Rule B-10.** `ntui` depends on the language, its prelude, and nothing else.
`[dependencies]` is empty and stays empty until a decision says otherwise.

That includes, specifically and deliberately:

- **not the compiler's `src/`.** `npkg` imports `../src/frontend/list.npk`
  because `npkg` lives in that tree; `ntui` does not, and reaching into a
  compiler's internals for a growable array would couple this library's
  correctness to a file whose header says it exists for the compiler's own
  tables.
- **not the compiler's `lib/`.** `lib/nio.npk`, `lib/nfs.npk`, `lib/nsys.npk`
  are on their way out of that repository into an `nlibc` sibling (the
  compiler's `LAYOUT.md` says so), so importing them today is importing a path
  that is scheduled to change. The syscall numbers `ntui` needs are few, they
  are x86-64 ABI facts, and they are declared in `src/term/sys.npk` with the
  same shape `nsys` uses. When `nlibc` exists as a package and dependency
  resolution works, replacing our copy with a dependency is a recorded decision
  and a small diff.
- **not terminfo, not ncurses, not any system database** (T-003;
  [`CAPABILITIES.md`](CAPABILITIES.md)).

**Rule B-11.** The prelude is fair game and is used heavily: `Duration`,
`Path`, `OwnedFd`, `Reader`/`Writer`, `ByteReader`/`ByteWriter`,
`TextWriter`/`LineBufWriter`, the flag families (`oflags`, `fmode`), the named
errnos (`WouldBlock`, `Interrupted`), `Ordering`, the seven derivable traits,
and the trap error identities. Every module has it bound with no import.

---

## 5. Storage primitives

**Rule B-12 (T-006).** `ntui` declares its own growable array,
`src/core/vec.npk`:

```nitpick
pub struct:Vec<T> = { wild T->:items; int64:count; int64:cap; };
```

It is the compiler's `List<T>` in shape — a `wild` block, a count, a capacity,
doubling through `#size_of<T>()` — because that shape is right and has been
exercised across twenty-two families. It is **ours** because a library must not
import a compiler's internals (B-10), and because ours will grow operations the
compiler's has no use for: `pop`, `truncate`, `insert`, `remove`, `swap_remove`,
`fill`, and a borrowing `at` for scalar elements.

Two more, both in `src/core/`:

- **`Bytes`** — an owning byte sink over `buffer`, with `push`, `extend`,
  `extend_str`, `put_uint` (decimal, no allocation), and `take`. Every escape
  sequence the renderer emits is composed into one of these. It exists because
  `string_concat` allocates per call and a frame is thousands of small pieces:
  the compiler measured exactly this shape as quadratic in `npkg`'s first full
  run, 17 minutes of 56 spent in the kernel.
- **`SmallMap<K, V>`** — a fixed-capacity association list for the handful of
  cases that need one (capability overrides, the cluster spill table). Not a
  hash map: the sizes are tens, a linear scan over a contiguous array beats a
  hash at that size, and there is nothing to verify.

**Rule B-13.** `ntui` declares no other container. A widget that wants a tree
takes an `arena<T>` and `Handle<T>` from the language, which is what they are
for.

---

## 6. Modules, files, and imports

**Rule B-14.** One module per file, and **a file's `mod:` name must equal its
basename** — the loader reports `NITPICK-RESOLVE-005` at line 1 otherwise, and
says nothing about the name.

**Rule B-15.** Public names carry the module's short prefix and nothing else
carries it: `term_enter`, `input_decode`, `cell_at`, `layout_split`. A
`pub struct` takes the ecosystem's PascalCase (`Terminal`, `Cell`, `Rect`).
Constants are `SCREAMING_SNAKE`. This is the prelude's own convention and the
compiler's.

**Rule B-16 — imports are relative today.** Until dependency roots are
populated (§1), every internal import is `use "./x.npk".*;` or
`use "../y/z.npk".*;`, and a **consumer** of `ntui` imports it by a relative
path to its `src/lib.npk`. `src/lib.npk` is the umbrella: it `pub use`s the
public surface so a consumer writes one import.

> **`use` is not transitive** (`MODULE_REFERENCE.md` §2.3): a symbol imported
> into a module is not re-exported. `src/lib.npk` therefore has to re-export
> deliberately, which is a feature — the public surface is a list in one file
> that a reviewer can read.

**Rule B-17 — a `use` cycle is legal** (D-086) and is still a decomposition
mistake. `ntui`'s layers are acyclic by construction, and the harness has a
check that says so, because the compiler's experience is that a layering
violation arrives as a cycle six months after somebody moved one function.

**Rule B-18 — the layering, and the direction of every arrow.**

```
        widget  ──►  screen ──►  style
           │           │  │        │
           │           │  └──►  unicode
           ▼           ▼           ▲
          app  ──►   layout        │
           │                       │
           ├──►  input  ───────────┘
           └──►  term   ──►  caps ──► core
                   │                    ▲
                   └────────────────────┘
```

`core` depends on nothing. `unicode` depends on `core`. Nothing depends on
`app` except an application. A module may not import a module to its left.

---

## 7. Reserved words that will bite

The compiler's list, filtered to the ones a terminal library will reach for and
with the ones this domain adds. Each of these reads like an ordinary local name
and is not:

| Wanted as a name | Actually |
|---|---|
| `fd` | one of the five kernel identifier **types** (D-042) |
| `buffer` | the owning byte cell **type** |
| `raw` | the unwrap keyword — and this library says "raw mode" constantly |
| `move`, `drop`, `pass`, `fail`, `relay`, `give`, `pick`, `fall` | keywords |
| `end` | the `when`/`then`/`end` terminator — and "end" is what a range wants to be called |
| `in` | the `for … in` keyword — and "in" is what an input byte source wants to be called |
| `mod` | the module keyword |
| `limit` | the verification keyword — and "limit" is what a constraint wants to be called |
| `any` | the type |
| `on` | a keyword — `Focus?:on = …` does not parse |
| `error` | the declaration keyword; `Result`'s field is `.err` |
| `thread`, `channel`, `atomic`, `joins`, `gives` | concurrency keywords |
| `unit` | the unit-declaration keyword |
| `trit`, `nit` | ternary type keywords (interned as names only after a `.`) |
| `oflags`, `prot`, `mflags`, `fmode` | the four flag-family type keywords |
| `Mutex`, `Guard`, `RwLock`, `RGuard`, `CondVar`, `Barrier`, `acquire` | sync primitive keywords |
| `is`, `is_err`, `defaults`, `as`, `with`, `where`, `never`, `fails` | keywords |

The names this library therefore uses instead, fixed here so they are used
consistently: `descr` for a raw descriptor number, `sink` for a byte
destination, `src`/`source` for a byte origin, `hi` for a range's upper bound,
`cap_set` for a capability set (`caps` for the value), `bound` for a layout
constraint, `mode_bits` for the restore mask.

---

## 8. Three more shapes that are not what a C or Rust habit expects

- **Adjacent string literals do not concatenate.** `"a" "b"` is two literals.
  A long escape sequence is built with `Bytes`, not by juxtaposition.
- **`discard(expr);` takes parentheses; `defer { … }` takes no trailing
  semicolon.** Both wrong forms are parse errors.
- **Struct/trait/impl/function declarations end `};`. Control-flow blocks do
  not.** A semicolon after an `if`'s closing brace is a syntax error.

---

## 9. Open items

- **O-B1 — when `npkg` can build a library.** The trigger to migrate is
  `npkg build` honouring `target = "library"` and `[dependencies]` populating
  the resolver's root list. Neither is on the compiler's 1.5 or 1.6 map, so
  this is a request to be made, not a date to wait for. Tracked in
  `../OPEN_QUESTIONS.md` as O-N2.
- **O-B2 — whether `ntui` ships as source or as an object.** A `.o` plus a
  declaration file would build faster; source keeps the closed-world link and
  the whole-program verification story intact, which is worth more. Recorded as
  settled-for-now in favour of source; revisit only if build times become a
  real complaint.
