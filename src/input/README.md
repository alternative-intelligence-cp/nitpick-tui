# `src/input/` — the decoder

A pure incremental function from bytes to events: no I/O, no descriptor, and a
clock supplied by the caller. Governed by `meta/specs/INPUT_MODEL.md`. Built in
cycle 0.3.
