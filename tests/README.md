# `tests/`

| Directory | Stage | Contents |
|---|---|---|
| `probe/` | `program` | the cycle-0.0 language probes; **never deleted** |
| `conformance/` | `accept` | the public API compiles in a program that only imports it |
| `unit/` | `program` | behaviour, judged by exit code |
| `golden/` | `golden` | render -> bytes -> mini-VT -> screen, three artifacts per case |
| `rejection/` | `check` | programs the compiler must refuse, with exactly the expected codes |
| `fixtures/` | `fixture` | captured terminal byte streams, byte-stable and committed |
| `vt/` | — | the mini-VT oracle; imports nothing from `src/` but `core` |

Expectations live in the test file. Governed by `../meta/specs/TESTING.md`.
