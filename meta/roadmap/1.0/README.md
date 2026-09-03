# Cycle 1.0 — Release

**Documentation, the API freeze, the `failsafe` arm contract, and versioning.**

## Decisions in

**T-103** — the version policy, settled in the second batch: `0.x` until the
compiler reaches 1.0 and the API has survived cycle 0.15's application, then
semantic versioning, and **adding a public error identity is a major version**.
This cycle *publishes* it; it does not decide it.

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| 1.0.0 | **The API freeze** — the public surface enumerated in one file, reviewed name by name | `src/lib.npk` as the contract |
| 1.0.1 | **The `failsafe` contract** — the generated per-module arm list, published | a consumer knows exactly what importing costs |
| 1.0.2 | **The guide** — `docs/`: getting started, the model, every widget, the terminal notes | a person can build an application from the documentation alone |
| 1.0.3 | **Examples** — one per widget, plus the three architectural ones | every example built and run by the harness |
| 1.0.4 | **Versioning** — T-103 published | the policy stated where a consumer will read it, including what a major version means |
| 1.0.5 | **Close** | `done/1.0/`, the post-1.0 map reviewed |

## Checklist

### 1.0.0 — the API freeze
- [ ] `src/lib.npk` lists every public name, one per line, grouped by module
- [ ] each reviewed: is it needed, is it named right, does it belong at this layer?
- [ ] anything not needed removed **now** — removing a public name after 1.0 is a major version
- [ ] a conformance test that imports the umbrella and touches every name, so a removal breaks a test rather than a user

### 1.0.1 — the `failsafe` contract
- [ ] the per-import arm list generated (S-10) and published in `docs/`
- [ ] `check_failsafe_arms` proving the published list is what a program actually owes
- [ ] the S-4 sentence — call `ntui_restore()` first in `failsafe` — stated once, prominently, and demonstrated in every example
- [ ] the error budget (nine) stated as part of the contract, with the rule that a tenth is a major version

### 1.0.2 — the guide
- [ ] getting started: a working program in under thirty lines
- [ ] the model: immediate mode, the loop, `update`/`view`, why there are no closures and what that means for you
- [ ] every widget, with a rendered screenshot generated **by the golden suite** so the documentation cannot show something the library does not produce
- [ ] the terminal notes: what `ntui` sets, what it restores, what it cannot restore (`SIGKILL`), and how to diagnose a misbehaving terminal with `ntui_caps_explain()` and `NTUI_CAPS`
- [ ] the compatibility matrix, as measured
- [ ] a page on what `ntui` deliberately does not do, and why: no terminfo, no bidi, no line breaking, no images at 1.0, no component tree

### 1.0.3 — examples
- [ ] one per widget, minimal
- [ ] three architectural: a multi-screen application, one with background work through the post channel, one that shells out and resumes
- [ ] every example built **and run** by the harness, so a broken example is a red run

### 1.0.4 — versioning
- [ ] T-103's policy written into `docs/` and into the repository's release notes
- [ ] the rule that **adding an error identity is a major version** stated prominently, because it breaks every consumer's compilation — a rule no other language's library has to have, and the one thing about `ntui` a consumer most needs to know
- [ ] the current error identity count (nine) published beside it, so a consumer can see the budget rather than infer it

## Gate

A person who has not seen the code can build a working application from
`docs/` alone, and every example is green in the harness.

## After

The post-1.0 map in `ROADMAP.md`: image protocols (1.1), an optional retained
layer (1.2), `aarch64` (1.3), and the verified build (1.4) once the compiler's
`npkg verify` reaches libraries.
