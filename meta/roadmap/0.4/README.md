# Cycle 0.4 — Capabilities

**`src/caps/`: the built-in table, the environment ladder, the runtime probe,
and the width calibration.** What the terminal can do, resolved once, and then
a constant for the life of the process.

## Why here

It needs the device (0.1) to write and the decoder (0.3) to read replies, and
everything above it needs its answers. It is also where T-003 — no terminfo —
stops being a decision and becomes code.

## Decisions in

T-003, T-024, T-034, and C-1 … C-14 in `specs/CAPABILITIES.md`. Settled.

Plus **T-107**: no DECRQM probe for `?1049`.

**Open by design:** O-C1, the table's initial contents — *data*, written here
against real terminals and completed at 0.16, not a preference anybody can
settle in advance.

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| 0.4.0 | **The table** — `tools/caps.toml`, `tools/gen_caps.py`, `src/caps/table.npk`, the default row | a table that regenerates byte-identically |
| 0.4.1 | **The environment ladder** — `NO_COLOR`, `CLICOLOR_FORCE`, `COLORTERM`, the identifying variables, multiplexer detection, `NTUI_CAPS` | every variable's effect asserted, and `NO_COLOR` unoverridable |
| 0.4.2 | **The probe** — the one write, DA1 as sentinel, the reply reader, the push-back rule | a terminal that answers nothing resolves to the table without hanging |
| 0.4.3 | **Calibration** — the four width probes through DSR, the erase, the fallback | four measured booleans on three real terminals, transcripts committed |
| 0.4.4 | **Explain** — `ntui_caps_explain()` and the per-field `CapSource` | one line per capability naming where the answer came from |
| 0.4.5 | **Close** | `done/0.4/`, `0.5.0.md` written |

## Checklist

### 0.4.0 — the table
- [ ] `tools/caps.toml` with a row per terminal, keyed by `TERM_PROGRAM` then `TERM`
- [ ] `tools/gen_caps.py` emitting `src/caps/table.npk`, committed, regeneration-checked
- [ ] the default row for an unrecognised `$TERM` containing `256color`: `Indexed256`, SGR mouse, bracketed paste, nothing else
- [ ] `Caps` is a plain value with no owning field except `term_name`, and the exception is justified in a comment
- [ ] `CapSource` recorded per field group

### 0.4.1 — the environment ladder
- [ ] the order in `specs/CAPABILITIES.md` §3, exactly
- [ ] `NO_COLOR` (set, non-empty) forces `color = None` and **nothing later re-enables it** (C-2) — asserted by a test that runs the probe afterwards
- [ ] `CLICOLOR_FORCE` never overrides `NO_COLOR`
- [ ] `COLORTERM=truecolor|24bit`
- [ ] `TMUX`, `STY`, `$TERM` prefixes → `multiplexed`
- [ ] `NTUI_CAPS` parsing, `name` and `no-name`, overriding everything
- [ ] a test per variable, run in a child with a controlled environment

### 0.4.2 — the probe
- [ ] the nine queries in one write, DA1 last (C-3)
- [ ] **DA1 is the sentinel**: absent replies mean unsupported, no per-query timeout, no heuristic (C-4)
- [ ] DECRQM `Pm` read as 1/3 = supported-set, 2/4 = supported-reset, 0 = not recognised (C-5)
- [ ] `?7`'s original state recorded into the restore record (TERMINAL_MODEL §7)
- [ ] the probe runs **before** the alternate screen is entered
- [ ] **the push-back rule** (I-27): keystrokes typed during startup are queued as events, in order, not discarded — asserted by a test that types before the probe
- [ ] the truecolour probe (C-6), run only when the table and environment disagree or are silent
- [ ] a terminal that answers nothing: the probe returns at its deadline with the table's answers, and the deadline is 100 ms not 100 seconds
- [ ] the probe never runs headless (C-8)

### 0.4.3 — calibration
- [ ] the four probe strings from `specs/CAPABILITIES.md` §5, each written at a known column with the cursor homed
- [ ] `CSI 6 n` after each, the reported column read from the `ReplyEvent`
- [ ] output on the **normal** screen, erased with `CR` + `CSI 2 K` (C-10)
- [ ] the four booleans wired into `cluster_width` (0.2's hooks)
- [ ] a terminal that does not answer leaves the static rules in place — a defined state, not a failure
- [ ] transcripts from three real terminals committed under `meta/research/captures/`
- [ ] `Caps.assume_narrow_zwj` as the application's override

### 0.4.4 — explain
- [ ] `ntui_caps_explain(Writer)` printing one line per capability: name, value, source
- [ ] the line for an `Application`-sourced field says so, so a bug report is unambiguous
- [ ] documented as the first thing to run when a terminal misbehaves

## Gate

The probe run against three real terminals with committed transcripts, and a
fourth that answers nothing resolving to its table row inside the deadline.

## Watch for

- **The probe reads events, not bytes** (C-7). Reaching into the descriptor
  directly here would be a second input path, and it is how the "it ate my
  first keypress" bug gets written.
- **Multiplexers answer for themselves** and that is correct (C-12). What needs
  care is not pushing kitty flags through one unless the probe confirmed them,
  and never attempting `DCS tmux;` passthrough.
- **`prot` and `fmode` are type keywords.** A capabilities module wants
  neither, but `caps` as a local shadows nothing and `cap_set` is the reserved
  spelling from `specs/BUILD.md` §7.
