# Cycle 0.15 — The dogfood application

**A real program, in `examples/`, written against the library as a consumer.**

## Why a cycle

Because an example written by the person who wrote the API is weak evidence.
The library's own examples demonstrate features; a program with a purpose finds
what is missing, what is awkward, and what is wrong — and it finds it before a
1.0 that would have to keep it.

This is **T-104**, as amended by **T-114**: the program lives in
[`nitpick-apps`](https://github.com/alternative-intelligence-cp/nitpick-apps),
the application workbench, not in this repository's `examples/`. A consumer is
a real program with its own lifetime, and `examples/` would make one that
outgrows this library move, and one that consumes three libraries pick a
parent.

**O-W4 is settled (T-115): its own repository**, not inside `nitpick-posix`.
A pager exercises the screen model and almost no widgets; this program has to
exercise the whole stack or the cycle buys nothing.

The import is by relative path to `../../nitpick-libs/nitpick-tui/src/lib.npk`
until the compiler's dependency resolution lands (O-N2), with a comment at the
site naming the open question.

## What to build

A **log viewer**: open a file or read standard input, follow it, search with a
pattern, filter, jump, mark, and show a status line. It is chosen deliberately
over a prettier demo because it exercises exactly the parts most likely to be
weak:

| Feature | Exercises |
|---|---|
| a very large file | `List`/`TextArea` performance, the scroll path, the 0.14 optimisation |
| follow mode | timers, the post channel, background work, redraw policy |
| search and filter | `TextInput`, key bindings, state the application owns |
| lines of arbitrary bytes | UTF-8 substitution, control-character rendering, width |
| terminal resize while following | the resize path under load |
| shelling out to `$PAGER` | suspend and resume, the O-N1 signal-mask window |
| running over SSH | the write path under latency, the escape-timeout default |

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| 0.15.0 | **The program** — written straight through, recording every friction | a working log viewer, and a numbered list of findings |
| 0.15.1 | **The findings, triaged** — each one a defect, a missing feature, or an accepted cost | a decision per finding |
| 0.15.2 | **The fixes** — the defects and the missing features that survive triage | the library changed, the golden suite re-recorded where the change was intended |
| 0.15.3 | **Close** | `done/0.15/`, `0.16.0.md` written |

## Checklist

### 0.15.0 — the program
- [ ] the repository created (T-115: its own), registered in `nitpick-apps`' `APPS.md`, with its GitHub description and topics set in the same pass
- [ ] written **without changing the library**, so that every friction is recorded rather than smoothed over as it appears
- [ ] every awkwardness written down as it is met, numbered, with the line of code that caused it
- [ ] runs over SSH, in `tmux`, in a terminal resized while following, and with a 1 GB log
- [ ] a `NTUI_RECORD` session of each committed as a replay test

### 0.15.1 — triage
- [ ] every finding classified: **defect** (the library is wrong), **gap** (the library is missing something a consumer reasonably needs), or **cost** (the library is right and this is what the design costs)
- [ ] every `cost` written into the documentation, because an accepted cost nobody warned about is a defect in the documentation
- [ ] every `gap` sized, and either scheduled into 0.16 or recorded as post-1.0
- [ ] **O-E2 revisited**: did this application want a `Router`? A decision either way, with the code that would have used it

### 0.15.2 — the fixes
- [ ] the defects fixed, each with a regression test
- [ ] the scheduled gaps closed
- [ ] the golden suite re-recorded only where a change was intended, and a re-record reviewed as a diff of screen dumps

## Gate

A log viewer that a person would actually use, and a triaged findings list with
a decision per entry.

## Watch for

- **The temptation to fix as you write.** The value of this cycle is the
  *record* of what was awkward, and a friction smoothed over in the moment is a
  friction the next user meets too.
- **"It needs a feature" is usually "the example needs a helper".** A gap is
  only a gap if it cannot be written in the application in a reasonable number
  of lines.
