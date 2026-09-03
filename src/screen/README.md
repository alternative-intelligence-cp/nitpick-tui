# `src/screen/` — cells, buffers, and the renderer

`Cell`, `Buffer`, the cluster pool, per-row damage, the drawing primitives, and
`screen_render` — the pure function at the centre of the library. Every escape
sequence the library emits is spelled in `seq.npk` and nowhere else. Governed
by `meta/specs/SCREEN_MODEL.md`. Built in cycles 0.6 and 0.8.
