# `src/` — the library

Nitpick source only. The layering and the direction of every dependency arrow
are in `../meta/specs/BUILD.md` §6; a module may not import one to its left.
`lib.npk` is the umbrella and lists the public surface, one name per line,
because `use` is not transitive.
