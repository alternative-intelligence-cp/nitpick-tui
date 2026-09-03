# Cycle 0.16 — Hardening

**The fuzz sweep, the stress sweep, the compatibility matrix, and the
verification obligations reconciled.** Everything the plan deferred to "before
release", collected.

## Decisions in

T-094, and `specs/VERIFICATION.md` in full.

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| 0.16.0 | **The fuzz sweep** — the input decoder, the UTF-8 decoder, the segmenter, the escape parser, and the golden byte parser | a hundred million inputs across all five, clean |
| 0.16.1 | **The stress sweep** — `// stress: 40` on everything with a timing dimension | forty runs, same answer, every time |
| 0.16.2 | **The compatibility matrix** — fixtures and capability transcripts for every terminal in `specs/COMPAT.md` §3 | every row Verified or removed |
| 0.16.3 | **The degradation matrix** — the combination test from K-6 | a golden per combination |
| 0.16.4 | **The verification reconciliation** — `specs/VERIFICATION.md`'s list against what the code generates | an obligation list that is true |
| 0.16.5 | **The audit** — a read of the whole tree against the specifications | a findings list, and the specifications corrected |
| 0.16.6 | **Close** | `done/0.16/`, `1.0.0.md` written |

## Checklist

### 0.16.0 — fuzz
- [ ] the input decoder: random bytes, structured fixture mutations, the four invariants
- [ ] the UTF-8 decoder: every byte sequence up to length 5, exhaustively for lengths 1–3 and sampled beyond
- [ ] the grapheme segmenter: random cluster sequences, asserting termination and boundary consistency with the UCD test file
- [ ] `Bytes` and `Vec<T>`: random operation sequences against a reference model
- [ ] the golden byte parser (the oracle): random bytes, asserting it either parses or refuses, and never traps
- [ ] every corpus that found something committed permanently

### 0.16.1 — stress
- [ ] `// stress: 40` on: the signal path, suspend/resume, the post channel, the escape timeout, the runtime's wait, and every device test
- [ ] a red under stress is a **stop sign, never a retry** — the compiler's R5, and the reason is that every timing-shaped defect it found looked like flakiness first

### 0.16.2 — the compatibility matrix
- [ ] a fixture directory and a capability transcript for every terminal claimed
- [ ] **a terminal with no fixtures is removed from the table**, not downgraded (T-094)
- [ ] `tmux` and GNU `screen` as rows in their own right, over at least two host terminals each
- [ ] the matrix's tier column filled honestly

### 0.16.3 — the degradation matrix
- [ ] one reference screen under every combination in K-6, each with a committed golden
- [ ] `NTUI_CAPS` proven to reproduce each combination, so a bug report is actionable

### 0.16.4 — verification
- [ ] `specs/VERIFICATION.md`'s obligation list read against the code, entry by entry
- [ ] every obligation the code generates that the list does not name, added
- [ ] every obligation the list names that the code does not generate, removed or scheduled
- [ ] the contracts written as comments (P-1) checked to be syntactically what they will be, by pasting one into a scratch file and confirming the compiler's rung refuses it **by name** rather than parsing it as something else
- [ ] the property tests standing in for each, present and green
- [ ] the whole list handed forward as `meta/OBLIGATIONS.md`, ready for the compiler's verified build (P-11)

### 0.16.5 — the audit
- [ ] every specification document read against the code that implements it
- [ ] every rule numbered in a specification either implemented, refused with a reason, or struck by a decision
- [ ] the tree checks' coverage reviewed: is there a document nothing diffs against?
- [ ] `check_specs_current`'s backlog drained

## Gate

Every tree check green, every claimed terminal backed by fixtures, and the
obligation list true.

## Watch for

- **The audit is the cycle's most valuable part and the easiest to shorten.**
  The compiler's cycle 0.6 found every one of its holes this way and none of
  them by a test.
- **A specification rule with no implementation and no refusal is the failure
  this cycle exists to find.** It is the dormant-rule pattern, and the compiler
  found it three times.
