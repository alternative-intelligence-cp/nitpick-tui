# Cycle 0.0 — Foundations

**The probes, the harness, and `src/core/`.** Nothing in this cycle draws
anything. What it produces is the ability to find out whether the rest of the
plan is buildable, and the machinery every later cycle is tested by.

## Why this shape

Two of the compiler project's most expensive lessons decide this cycle's
contents:

- **"A construct that parses is not a construct that works."** Its cycle 0.4
  was mostly repair, and every repair dated to the cycle that had parsed the
  construct. `ntui`'s design leans on several language shapes that have never
  been exercised in this combination — a struct holding a borrow built by
  literal and passed down, a 32-byte POD in a large array, an inherent impl on
  a generic struct, an `async` trait method reached through a generic bound.
  **0.0.0 asks the compiler about all of them before anything is built on
  them.** A probe that fails changes the design; finding that out now costs a
  day, and finding it out in 0.11 costs a cycle.
- **"Diagnostics come first, not last — they are how every later cycle is
  tested."** Here that is the harness. A suite written after the code is a
  suite shaped by the code.

## Decisions in

T-001, T-002, T-005, T-006, T-007, T-008, T-056, T-092, T-093. All settled.
**Nothing in this cycle is blocked on a question.**

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| [0.0.0](0.0.0.md) | **The language probes** — fourteen programs asking the compiler whether the design is spellable | a recorded verdict per probe, and any design change the answers force |
| [0.0.1](0.0.1.md) | **The skeleton** — `nitpick.toml`'s test table, the module layout, `src/lib.npk`, CI | `npkc` compiles an empty library and a program that imports it |
| [0.0.2](0.0.2.md) | **The harness, part 1** — build, the `program` stage, the toolchain pin | one test program builds, links, runs, and its exit code is judged |
| [0.0.3](0.0.3.md) | **The harness, part 2** — `parse`, `accept`, `check`, `device`; the self-check; the tree checks | the self-check proves the harness can fail, seven ways |
| [0.0.4](0.0.4.md) | **`src/core/`** — `Vec<T>`, `Bytes`, `SmallMap<K,V>`, `limits.npk` | the three primitives, with their own suites and their obligations written |
| [0.0.5](0.0.5.md) | **Close** — the cycle's findings, the spec amendments the probes forced, the handoff to 0.1 | `done/0.0/`, and 0.1 openable by a fresh session |

## Checklist

### 0.0.0 — the probes
- [ ] `probe/probe01_borrow_struct.npk` — a struct holding `Buffer->`, built by literal, passed down two levels, **not** returned (T-056)
- [ ] `probe/probe02_borrow_return.npk` — the same struct **returned** from a function; **expected to be refused**, with the code recorded
- [ ] `probe/probe03_pod_array.npk` — a `Vec<Cell32>` of 12 000 32-byte POD values: fill, read, copy, clear
- [ ] `probe/probe04_payload_enum.npk` — a tagged enum with `Char(char32)` and `F(uint8)` payloads, destructured in a `pick`, stored in a fixed array
- [ ] `probe/probe05_generic_move.npk` — `move T:v` into a generic container, with `T` both scalar and owning
- [ ] `probe/probe06_inherent_generic.npk` — `impl:<T>:Vec<T> = { … }` with a `Vec<T>->:self` receiver that mutates
- [ ] `probe/probe07_trait_selfptr.npk` — a trait with a `Self->` receiver, implemented on two structs, called through a generic bound
- [ ] `probe/probe08_async_trait.npk` — an `async` trait method driven through a generic bound and through `dyn`
- [ ] `probe/probe09_sys_ioctl.npk` — a `buffer`-laid-out 36-byte structure handed to `ioctl(TCGETS)` through `sys`, read back field by field
- [ ] `probe/probe10_sys_eventfd.npk` — `eventfd2`, `io_watch`, one `suspend_io` over two descriptors, a write and a wake
- [ ] `probe/probe11_signalfd.npk` — `rt_sigprocmask` + `signalfd4`, a self-sent `SIGWINCH`, the 128-byte record read back
- [ ] `probe/probe12_static_state.npk` — module-level `fixed` state with a mutable interior reached through an `atomic<int32>`, written from `failsafe`
- [ ] `probe/probe13_failsafe_restore.npk` — a deliberate trap with a "restore" writing to a descriptor from `failsafe`, proving the trap path reaches it
- [ ] `probe/probe14_string_bytes.npk` — `string_bytes` into a decoder, `string_from_bytes` back, with the borrow rules exercised at every edge
- [ ] a verdict line per probe recorded in `0.0.0.md`, with the exact diagnostic where one was refused
- [ ] every design consequence written into `meta/specs/` **and** `meta/DECISIONS.md` before 0.0.1 starts

### 0.0.1 — the skeleton
- [ ] `src/lib.npk` exists and `pub use`s nothing yet (`MODULE_REFERENCE` §2.3: `use` is not transitive, so the surface is a deliberate list)
- [ ] every `src/` subdirectory has a placeholder module that parses, so the `parse` stage has something to sweep
- [ ] `nitpick.toml`'s `[[test]]` table has its first entry
- [ ] a consumer program under `tests/conformance/` imports `src/lib.npk` by relative path and compiles
- [ ] CI: a workflow that runs `harness/run.py` on push, with the LLVM 20.1.2 toolchain and the compiler built from a pinned commit
- [ ] `CONTRIBUTING.md` and `CLAUDE.md` re-read against 0.0.0's verdicts and extended (both were written at repository setup; this is the check that they are still true)

### 0.0.2 — the harness, part 1
- [ ] `harness/run.py`: the manifest reader, the toolchain pin check, the module-graph walk
- [ ] the build pipeline — `npkc` → `opt` (check leg) → `llc` → the undefined-symbol scan → `ld.lld`
- [ ] the undefined-symbol scan against the runtime allowlist, as a **build step** and not a test (B-2)
- [ ] the `program` stage, at -O0 and again under `opt -O2`, same exit required (B-3)
- [ ] `// expect-exit:` and `// stress: N` honoured
- [ ] the `repro` check: two builds from different working directories, byte-identical IR (B-4)
- [ ] one real test program green

### 0.0.3 — the harness, part 2
- [ ] the `parse` stage over every `.npk` in the tree, each file once
- [ ] the `accept` and `check` stages, with the **exact-code** rule (B-7 / T-092)
- [ ] the `device` stage, skipped **loudly** with its reason when no terminal is available (V-3)
- [ ] `--only`, and output that says twice that a filtered run concludes nothing
- [ ] `harness/selfcheck.py` with all seven cases from `specs/TESTING.md` V-14
- [ ] the self-check runs **first** in every full invocation
- [ ] `check_layering` — every `use` edge against `specs/BUILD.md` §6's diagram
- [ ] `check_constants_named` and `check_no_owning_cells` (they will have nothing to check yet, and that is the right answer, not a reason to skip)
- [ ] `check_error_budget` reading `specs/SAFETY.md` §3's table

### 0.0.4 — `src/core/`
- [ ] `src/core/limits.npk` — every bound from `specs/INPUT_MODEL.md` §13 and every named constant from `specs/SAFETY.md` §4, in one file
- [ ] `src/core/vec.npk` — `Vec<T>`: `init`, `reserve`, `push`, `pop`, `at`, `set`, `truncate`, `clear`, `insert`, `remove`, `swap_remove`, `fill`, `free`
- [ ] `Vec<T>` exercised at both `T` shapes: a scalar and an owning value with `move T:v`
- [ ] `src/core/bytes.npk` — `Bytes`: `init`, `push`, `extend`, `extend_str`, `put_uint`, `put_int`, `len`, `view`, `take`, `clear`, `free`
- [ ] `put_uint` allocation-free and correct at 0, 1, 9, 10, 99, 100, and `uint64` maximum
- [ ] `Bytes` growth is amortised linear, proven by a test that appends a million bytes and bounds the reallocation count — the compiler's own quadratic-capture defect is why this test exists
- [ ] `src/core/smallmap.npk` — `SmallMap<K,V>` over a fixed capacity, with `ETuiCapacity` on overflow
- [ ] every `Vec` and `Bytes` accessor's bounds obligation written as a comment in the `requires`/`ensures` syntax it will take (`specs/VERIFICATION.md` P-1)
- [ ] property tests standing in for each written obligation

### 0.0.5 — close
- [ ] every probe verdict recorded, every forced spec amendment landed
- [ ] the harness self-check green, the full run green
- [ ] `meta/DECISIONS.md` updated with anything the probes settled
- [ ] `0.1/0.1.0.md` written execution-grade before the cycle closes
- [ ] cycle moved to `done/0.0/`

## Gate

**The cycle is complete when**: a full `harness/run.py` is green; the
self-check proves the harness fails seven ways; `src/core/`'s three primitives
each have a suite; and every probe has a recorded verdict with its
consequences written into the specifications.

## Watch for

- **A probe that fails is a finding, not an obstacle.** Record the exact
  diagnostic, decide the design change, amend the specification, and only then
  continue. Working around a compiler refusal in library code is what the
  compiler's own R6 forbids, for exactly the reason it applies here: a
  workaround buried in library code outlives the reason for it.
- **The reserved words** in `specs/BUILD.md` §7 will bite in `src/core/`
  specifically: `buffer`, `raw`, `move`, `end`, `in`, `limit` and `any` are all
  words a container library reaches for.
- **`Vec<T>` is `wild` storage** and every path out of a function that took some
  must release it, or `exit 0` traps under D-151. The suite's programs exit 0
  on purpose so that a leak on any path turns a pass into a trap.
