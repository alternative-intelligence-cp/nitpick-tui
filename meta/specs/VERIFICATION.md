# Verification obligations

The compiler's cycle 1.5 makes `prove`, `requires`/`ensures`, `limit<Rules>`
and Z3 real. Its orchestration rules say that **every branch records its own
verification obligations and the orchestrator merges them** (R9), because
parallel authorship of library code is parallel authorship of proof
obligations, and obligations discovered in a branch and never collected are the
cheapest way to lose the campaign.

This document is `ntui`'s list. It is written **before** the code, kept current
as the code lands, and is the input `ntui` hands to the compiler's obligation
manifest when the verified build reaches libraries.

---

## 1. Where this stands

| Compiler subcycle | What it gives us | Our state |
|---|---|---|
| 1.5.0 (done) | the SMT writer, z3 under a pinned profile, the obligation manifest, `llvm.assume` elision | the division and bounds obligations `ntui` generates are already decidable |
| 1.5.1 | `limit<R>` names resolve, `Rules` bodies type, contract expressions type | §5's `limit` types become writable |
| 1.5.2 | `limit<Rules>` live | §5 lands |
| 1.5.3 | contracts live | §4's `ensures` clauses land |
| 1.5.4 | `prove` / `assert_static` | §6's inline proofs land |

**Rule P-1.** Until a construct is live, its obligation is stated **as a
comment beside the code in the exact syntax it will take**, and is enforced by
a property test. That way the switch is deleting a comment marker rather than
inventing the clause, and the property test is what holds the line in the
meantime. The compiler's own rungs refuse the constructs by name today, so a
premature `ensures` is a build failure, not a silent no-op.

---

## 2. What the language discharges for free

Worth stating, because it is most of the list and it is why the residue is
small:

- **Every index into a slice, array or `buffer` is bounds-checked** and traps
  (D-070). `ntui` never reads out of bounds; the question is only whether a
  *reachable* index is out of bounds, which is what §3 is about.
- **Every plain integer `+ - *` traps on overflow** (D-210). Layout arithmetic
  cannot silently produce a wrong rectangle.
- **Division by zero and `MIN / -1` trap** (D-007).
- **Borrows cannot escape** (D-004), so a `Surface` cannot outlive its buffer.
- **Owning values are move-only** (TYPE-046), so no cell grid is aliased.
- **`Result<T>` everywhere** with no unchecked unwrap outside a `never fails`
  callee (D-163), so no error is dropped.

The obligations below are the ones that remain: the ones where a trap is a
crash we would rather prove cannot happen, and the ones that are properties of
`ntui`'s own model rather than of the language.

---

## 3. Bounds — the largest class

**Rule P-2.** Every buffer index goes through `buffer_at` / `buffer_set`
(`SCREEN_MODEL.md` R-7), which is one function pair. Its obligation:

```nitpick
func:buffer_index = int64(Buffer->:b, uint16:row, uint16:col)
    requires (row < b.rows && col < b.cols)
    ensures  (result >= 0i64 && result < b.cells.count)
    never fails { … };
```

Discharging it makes **every cell access in the library** safe by construction
rather than by a runtime check at each site — and elides the runtime check,
which is the D-218.9 payoff: the renderer's inner loop loses a compare and a
branch per cell.

The callers then owe `row < rows && col < cols`, which is where the clipping
in `SCREEN_MODEL.md` §9 does its work: a `Surface` write is clipped before it
becomes an index, so the caller's obligation is discharged by the clip.

| Site | Obligation | How discharged |
|---|---|---|
| `buffer_index` | index in range | contract, Z3 |
| `surface_set_cluster` | the clip establishes the contract's precondition | contract, Z3 |
| `Vec<T>` `at`/`set` | index `< count` | contract, Z3 |
| cluster pool read | offset + length `<= pool.len` | contract, Z3 |
| range-table binary search | `lo <= hi` maintained; the loop terminates | invariant + variant, Z3 |
| link table lookup | `id < links.count`, and `id != 0` | contract, Z3 |

---

## 4. Layout — the six invariants

`LAYOUT_MODEL.md` L-5's six, as `ensures` on `layout_split`:

```nitpick
pub func:layout_split = Vec<Rect>(Layout->:l, Rect:area)
    ensures (result.count == l.parts.count)
    ensures (forall_inside(result, area))        // L-5.2
    ensures (no_overlap(result))                 // L-5.3
    ensures (ordered_along(result, l.dir))       // L-5.4
    ensures (cross_axis_equals(result, area, l.margin))
    ensures (total_fits(result, area, l.gap))    // L-5.6
{ … };
```

**Rule P-3.** Four of the six are quantified over the result, which is a
harder shape for Z3 than a scalar. The plan's approach: prove them for the
**generated** structure rather than by quantifying — the solver sees the
recurrence `pos_{i+1} = pos_i + size_i + g` and the per-step facts
`size_i >= 0` and `pos_i + size_i <= area_end`, from which the four follow by
induction on the loop's invariant. If that turns out not to discharge, the
fallback is a runtime check in a debug build plus the property test, recorded
as an open row in the manifest rather than as a silent assumption — which is
what `open` verdicts are for.

**Rule P-4.** The percentage and ratio arithmetic:

```
requires (p <= 100)                       // Percent
requires (b != 0)                         // Ratio; b == 0 is answered, not divided
ensures  (result <= avail)
```

The last one is the interesting one: it is what makes `total_fits` provable
without reasoning about the division.

---

## 5. `limit<Rules>` — the geometry types

**Rule P-5.** When 1.5.2 lands, three types become `limit`ed and the checks
inject at initialisation, at every assignment, and at parameter entry:

```nitpick
Rules:CellCoord = { $ >= 0u16; $ < 65535u16; };
Rules:Percent   = { $ <= 100u16; };
Rules:Utf8Len   = { $ >= 1u8; $ <= 4u8; };
```

The value of doing it here rather than by hand-written checks: a `limit`ed
parameter's precondition is discharged at the *caller* where the caller's own
knowledge proves it, and retained as a runtime check only where it cannot be —
and the manifest records which is which, per site.

---

## 6. `prove` sites

**Rule P-6.** Inline `prove(…)` where a local fact is cheap to state and
expensive to lose:

| Site | Proof |
|---|---|
| after a UTF-8 decode | the consumed length is 1…4 and the codepoint is a scalar value (not a surrogate, `<= 0x10FFFF`) |
| after a width lookup | the result is 0, 1 or 2 — nothing else, ever |
| after a grapheme step | the cluster end is strictly greater than the start, so the segmenter's loop terminates |
| in the renderer's cell loop | the tracked cursor column equals the sum of the widths emitted |
| in the diff | every damaged span is within its row |
| after degradation | an `Indexed` result is `< 256`, an `Ansi` result is `< 16` |

**Rule P-7 — the grapheme segmenter's termination is the one worth naming.**
It is a loop over a byte slice driven by a state machine, and "always advances"
is exactly the property a hand-written segmenter loses when somebody adds a
rule that can return zero progress. A `prove` there turns a hang into a
compile error.

---

## 7. Termination

**Rule P-8.** Every loop in `ntui` outside the runtime's own event loop is
bounded by a value that decreases, and the bound is stated:

| Loop | Variant |
|---|---|
| UTF-8 decode | bytes remaining |
| grapheme segmentation | bytes remaining (P-7) |
| range-table binary search | `hi - lo`, halving |
| renderer row walk | rows remaining |
| renderer cell walk | columns remaining |
| layout distribution | segments remaining |
| write retry | the monotonic deadline |
| input decoder | bytes fed |

**Rule P-9.** The event loop does **not** terminate, by design, and its
obligation is different: every *iteration* terminates, and every wait is
bounded (`EVENT_MODEL.md` E-7, including the idle timeout that exists precisely
so this can be said).

---

## 8. What cannot be proven, and is stated instead

**Rule P-10 — the honest claim.** Following the compiler's TCB doctrine
(`TCB.md`, D-218.11: *verified middle-end plus validated floor*), `ntui`'s
verification claim covers **its own arithmetic and memory discipline**, and
does not and cannot cover:

- **the kernel.** `ioctl(TCSETS)` does what the kernel does.
- **the terminal.** Every capability is a claim the terminal made about itself,
  and a terminal that lies produces a wrong screen. The calibration
  (`TEXT_MODEL.md` §4) narrows this and does not close it.
- **the Unicode data.** The tables are generated from the UCD; their
  *invariants* are checked (sorted, disjoint) and their *contents* are the
  Consortium's.
- **`llc` and `ld.lld`**, which the compiler names as trusted components.

The residue is enumerated here rather than mitigated, which is the seL4
precedent the compiler cites and the only honest shape for a claim of this
kind.

---

## 9. The handoff

**Rule P-11.** When the compiler's verified build reaches libraries, `ntui`
hands over: this document's obligation list, the `nitpick.obligations` rows its
own build produces, and the property tests that stood in for each unproven row.
The plan's closing cycle owns that handoff, and R9 is why it is a deliverable
rather than a hope.
