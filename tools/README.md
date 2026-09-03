# `tools/` — generators and instruments

`gen_unicode.py` (the width and segmentation tables), `gen_caps.py` (the
capability table), `capture.npk` (terminal fixture capture), `fuzz_input.py`.
Everything a generator emits is **committed as source** and checked by
regeneration — a hand-edited generated file is the failure that prevents.
