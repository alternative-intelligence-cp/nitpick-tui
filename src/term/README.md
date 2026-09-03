# `src/term/` — the device

`/dev/tty`, termios, the window size, `signalfd`, the restore record, suspend
and resume, and the headless device. The only module that touches the kernel.
Governed by `meta/specs/TERMINAL_MODEL.md`. Built in cycle 0.1.
