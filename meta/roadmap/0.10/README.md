# Cycle 0.10 — The runtime

**`src/app/`: the `Program` trait, `Ctx`, the loop, timers, the post channel,
and `run`.** The first cycle whose output an application can hold.

## Decisions in

T-004, T-070 … T-074. Settled.

**Open questions to settle:** O-E1 (may `view` fail? recommendation: yes, with
widgets substituting rather than failing).

## Subcycles

| # | Topic | Ends with |
|---|---|---|
| 0.10.0 | **The traits** — `Program`, `Ctx`, `RunOpts`; the shapes probes 07 and 08 verified | a `Program` implemented twice and driven generically |
| 0.10.1 | **The wait** — `io_watch` over tty, signal fd and eventfd; one `suspend_io`; the deadline | one park, three descriptors, no `select` |
| 0.10.2 | **The loop** — drain, order, update, redraw, render, swap | `specs/EVENT_MODEL.md` §3's pseudocode, exactly |
| 0.10.3 | **Timers** — the fixed sorted array, absolute deadlines, one-shot | thirty-two timers, the thirty-third refused |
| 0.10.4 | **The post channel** — the eventfd, the bounded queue, `ntui_post` | a background task's message reaching `update` |
| 0.10.5 | **Redraw policy and the frame cap** — `always_redraw`, `max_fps`, coalescing | a burst of twenty events producing one frame |
| 0.10.6 | **Replay** — the recorder, `harness/replay.py` | a recorded session replaying frame for frame |
| 0.10.7 | **Close** | `done/0.10/`, `0.11.0.md` written |

## Checklist

### 0.10.0 — the traits
- [ ] `Program` with `async func:update` and sync `func:view` (T-070)
- [ ] `view` takes `(Buffer->, Rect)`, **not** a returned `Surface` (E-2, T-056)
- [ ] `Ctx` with exactly the requests in `specs/EVENT_MODEL.md` §2 — a closed set (T-071)
- [ ] `run<P: Program>` generic and statically dispatched
- [ ] `RunOpts`: `always_redraw`, `max_fps`, the three deadlines, the mouse mode, whether to enter the alternate screen
- [ ] O-E1 decided and recorded

### 0.10.1 — the wait
- [ ] `io_watch` on the tty, the signal fd and the eventfd; **one** `suspend_io` (E-5)
- [ ] `next_deadline()` = min(next timer, next frame when a redraw is pending, the escape timeout when a lone `ESC` is held), else `NTUI_IDLE_TIMEOUT` (E-7)
- [ ] **no unbounded wait anywhere** — the idle timeout exists so that can be said
- [ ] every watched descriptor's `io_unwatch` on every exit path
- [ ] `// stress: 40` on the wait tests

### 0.10.2 — the loop
- [ ] the order in E-17: signals, terminal input in byte order, posted messages in post order, timers by deadline then id
- [ ] all events processed, **then** one render (E-6)
- [ ] `buffer_clear(back)` → `view` → `screen_render` → one write → swap
- [ ] an error from `update` or from the device stops the loop, restores, and is returned from `run` (E-19, E-20)
- [ ] the terminal restored on every exit path, including the error paths, and `ntui_restore()` reached from the `Terminal`'s drop
- [ ] the first end-to-end program: a headless `Program` that counts events and quits, asserting the exact frame sequence

### 0.10.3 — timers
- [ ] a fixed 32-entry array sorted by deadline; the thirty-third is `ETuiCapacity`
- [ ] absolute `CLOCK_MONOTONIC` deadlines computed once at `set_timer` (E-13)
- [ ] one-shot only; a repeating timer is the application re-arming (E-11)
- [ ] `clear_timer`, and clearing one that does not exist is a no-op
- [ ] a test that arms thirty-two at scattered deadlines and asserts the exact fire order

### 0.10.4 — the post channel
- [ ] `eventfd2(0, EFD_CLOEXEC | EFD_NONBLOCK)`, watched in the loop's `suspend_io`
- [ ] a mutex-guarded fixed-capacity queue of 32-byte POD messages
- [ ] `ntui_post` appends and writes eight bytes
- [ ] a full queue returns `ETuiCapacity` — **it does not block and does not drop silently** (E-16)
- [ ] a test with a background task posting from another thread, `// stress: 40`

### 0.10.5 — redraw policy
- [ ] a redraw when the application asked, on `Resize`, on resume, and on the first frame (E-8)
- [ ] `always_redraw` default **on** (T-073), with the reasoning in a comment
- [ ] `max_fps` bounding emission, not processing (E-10)
- [ ] a burst of twenty scroll events producing one frame — asserted
- [ ] an idle application emitting zero bytes over a hundred wakes — asserted

### 0.10.6 — replay
- [ ] `NTUI_RECORD=path` writing `(wake_time_ns, source, bytes)` records
- [ ] `harness/replay.py` feeding one to a headless run and comparing every frame
- [ ] a recorded session from the 0.10.2 program replaying identically
- [ ] documented in `CONTRIBUTING.md` as how a bug report becomes a test

## Gate

A headless `Program` driven by a recorded session replays frame for frame, and
an idle application emits zero bytes.

## Watch for

- **Probe 10's verdict decides 0.10.1 and 0.10.4.** If one `suspend_io` cannot
  cover three descriptors, stop and re-plan rather than polling.
- **`thread`, `channel`, `atomic`, `joins` and `gives` are keywords** and this
  is the module that wants them.
- **A spawned task cannot outlive its scope** (D-062). Work that must survive
  across events is spawned in the application's `main`, not in `update`, and
  the examples have to show that clearly or every user will get it wrong once.
