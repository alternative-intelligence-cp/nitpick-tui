# The runtime: the loop, timers, and the application shape

---

## 1. The shape, and why it is this one

**T-004: immediate-mode rendering with an Elm-shaped loop.**

Nitpick has no closures (D-018). That single fact removes the architecture most
TUI toolkits use — a retained tree of widget objects wired together by callbacks
— because there is nothing to store in a callback slot but a bare function
pointer with no captured state, and a widget system built on those is a widget
system where every handler needs its context passed by hand.

What remains, and what fits:

- **The application owns its state.** Not the library. There is no widget tree
  to keep in sync with a model, because there is no widget tree.
- **Events flow one way**: the terminal produces them, the runtime orders them,
  `update` consumes them and mutates the application's own struct.
- **`view` is a pure function of the state onto a buffer**, called after
  `update`, drawing everything from scratch.

The costs are real and are accepted: the whole screen is composed every frame
(the renderer's diff is what makes that cheap — `SCREEN_MODEL.md` §6, and an
unchanged frame emits zero bytes by R-21), and a widget cannot hold state
across frames unless the application holds it for it (`WIDGET_MODEL.md` §4 says
how the stateful ones do).

---

## 2. The `Program` trait

```nitpick
pub trait:Program = {
    // Consume one event. May await. Mutates the application's own state.
    async func:update = NIL(Self->:self, Event:ev, Ctx->:ctx);

    // Draw the current state. Synchronous, pure, and called once per frame.
    func:view = NIL(Self->:self, Buffer->:buf, Rect:area);
};

pub async func:run<P: Program> = NIL(P->:app, Terminal->:t, RunOpts:opts);
```

**Rule E-1 — `update` is `async` and `view` is not.** An application waiting on
a file, a socket or a subprocess does it inside `update` with an ordinary
`await` and needs no command indirection to express it. `view` may not: drawing
is a pure function of state, calling it must not suspend, and a `view` that
could await would be a `view` that could see a half-updated model.

**Rule E-2 — `view` takes the buffer and a rect, not a `Surface`.** A
`Surface` (`SCREEN_MODEL.md` §9) is a struct holding a `Buffer->` borrow, and a
function *returning* one launders a borrow upward, which the escape analysis
refuses (D-004). Sub-surfaces are therefore built by **struct literal** at the
call site — `Surface{ buf: s.buf, area: child }` — never by a function that
returns one. This is a language constraint, it shapes the whole drawing API,
and it is verified by a probe program in cycle 0.0 before anything is built on
it.

**Rule E-3 — `Ctx` is how the application talks to the runtime.** It is a
borrow of a small struct with a fixed set of requests, and it is the reason
`Cmd` is not an enum the application extends:

```nitpick
pub struct:Ctx = { … };            // opaque; the methods are the API

ctx.quit(code)                     // finish the loop; `code` becomes run's answer
ctx.redraw()                       // draw next tick even if nothing changed
ctx.set_cursor(col, row)           // request the cursor here at frame close
ctx.hide_cursor()
ctx.set_cursor_style(CursorStyle)
ctx.set_title(text)                // pushes on first use, pops at teardown
ctx.set_mouse(MouseMode)           // Off | Buttons | Drag | AnyMotion
ctx.set_timer(id, Duration)        // a Tick with this id after the duration
ctx.clear_timer(id)
ctx.suspend()                      // TERMINAL_MODEL.md §8; resumes on return
ctx.bell()
```

**Rule E-4 — every one of these is a *runtime* concern.** Application work does
not go through `Ctx`; it goes in `update`'s body or in a task
(§6). The closed set is what makes `Ctx` expressible without closures, and
adding to it is a decision because each entry is a thing the runtime must know
how to do.

---

## 3. The loop

```
enter terminal, probe capabilities, calibrate widths      TERMINAL_MODEL, CAPABILITIES
loop:
    wait for readiness on { tty, signalfd, wakefd } with deadline = next_deadline()
    drain tty      -> input_feed  -> events
    drain signalfd -> Resize / Signal events
    drain wakefd   -> posted messages -> events
    expired timers -> Tick events
    for each event, in order:  await app.update(event, ctx)
    if a redraw is due:
        buffer_clear(back)
        app.view(back, full_area)
        bytes = screen_render(front, back, caps, cursor)
        write(bytes)                                       one write, SCREEN §6
        swap(front, back)
until ctx said quit
restore terminal
```

**Rule E-5 — one wait, several descriptors, no `select`.** The language has no
`select` across channels (D-072), and does not need one here: `io_watch` arms
each descriptor and a single `suspend_io` parks the task until any of them is
ready or the deadline passes. That is the mechanism the compiler's own Bridge
uses to wait on three descriptors at once, and it is the reason the runtime's
wait is a single suspension rather than a poll.

**Rule E-6 — events are processed in arrival order, and rendering happens
after all of them.** A burst of twenty scroll events produces one frame, not
twenty. This is the whole reason the loop is shaped as drain-then-render, and
it is what keeps a fast-repeating key from making the program feel slower the
faster you press it.

**Rule E-7 — `next_deadline()` is the minimum of**: the next application timer,
the next frame deadline when a redraw is pending and the frame cap is on (§5),
and the escape timeout when the input decoder is holding a lone `ESC`
(`INPUT_MODEL.md` §6). Nothing else. If none apply the wait is unbounded — and
"unbounded" here still means a `Duration` the runtime supplies, because
`SAFETY.md` S-11 admits no unbounded wait; it is a long one
(`NTUI_IDLE_TIMEOUT`, 60 s) after which the loop wakes, does nothing, and waits
again. A wait that can never return is a program that cannot be shown to
terminate.

---

## 4. Redraw policy

**Rule E-8.** A redraw happens when: the application asked (`ctx.redraw()`), a
`Resize` was processed, the terminal was resumed, or it is the first frame. It
does **not** happen merely because an event arrived — a mouse-move event that
changes nothing should cost nothing.

**Rule E-9.** `RunOpts.always_redraw` (default **on**) makes every event batch
end in a redraw. The reason the safer default is on: `ctx.redraw()` is a thing
an application forgets, and the failure — a screen that does not update — is
much more confusing than a wasted diff that emits zero bytes (R-21). An
application that has measured its way to caring turns it off.

**Rule E-10 — the frame cap.** `RunOpts.max_fps` (default 120) bounds how often
a redraw is *emitted*, not how often events are processed. When a redraw is due
sooner than the cap allows, the loop sets a deadline and coalesces.

---

## 5. Timers

**Rule E-11.** Timers are identified by an application-chosen `uint32` and fire
as `Event.Tick(id)`. They are **one-shot**; a repeating timer is an application
that re-arms in its handler, which is one line and removes the "did my repeating
timer drift" question.

**Rule E-12.** The runtime holds at most `NTUI_MAX_TIMERS` (32) of them in a
fixed array sorted by deadline. Setting a thirty-third fails `ETuiCapacity`. A
fixed array because thirty-two is more than any real application uses, a sorted
insert over thirty-two entries is faster than a heap, and a fixed array has no
allocation on the loop's hot path.

**Rule E-13.** Deadlines are absolute `CLOCK_MONOTONIC` nanoseconds computed
once from the relative `Duration` at `set_timer`, per D-176. A re-arm cannot
drift because nothing re-derives a deadline from a duration twice.

---

## 6. Background work

**Rule E-14.** A task spawned from `update` cannot outlive the scope that
spawned it (D-062), so it cannot outlive `update`. Work that must continue
across events is spawned in the scope that called `run` — that is, in the
application's `main` — and communicates back through the runtime's **post
channel**.

**Rule E-15 — the post channel is an `eventfd` plus a queue.** The runtime
creates an `eventfd2(0, EFD_CLOEXEC | EFD_NONBLOCK)` (syscall 290) and watches
it in the loop's `suspend_io`. `ntui_post(handle, msg)` appends to a
mutex-guarded fixed-capacity queue and writes eight bytes to the eventfd; the
loop drains the queue into `Event.Posted(msg)` events.

An `eventfd` rather than a channel because the loop's wait is over descriptors
(E-5) and a channel is not one; a mutex-guarded array rather than a channel
because the queue is bounded, the critical section is one copy, and the wake is
the eventfd's job. The message is a fixed 32-byte POD value the application
casts — the same shape a `Cell` uses and for the same reason.

**Rule E-16.** A full post queue does **not** block the poster and does not
drop silently: `ntui_post` returns `ETuiCapacity`. A background task that can
outrun the UI has a design question to answer, and the library's job is to
make it visible rather than to buffer without bound or to lose data.

---

## 7. Ordering

**Rule E-17 — the order within one wake is fixed**, so a replay test replays:

1. signals (a resize must be seen before the events that follow it are drawn),
2. terminal input, in byte order,
3. posted messages, in post order,
4. expired timers, in deadline order and then id order.

**Rule E-18 — the whole loop is replayable.** Given a recorded sequence of
`(wake_time, source, bytes)` the runtime produces the same events, the same
`update` calls, and the same frames. `TESTING.md` §6 is the mechanism, and E-17
is what makes it possible.

---

## 8. Errors from `update`

**Rule E-19.** `update` returns `Result<NIL>`, like every function. An error
returned from it **stops the loop**, restores the terminal, and is returned
from `run` — it is not swallowed and it is not logged. An application that
wants to survive its own errors handles them in `update`; a library that
decided for it would be deciding whether a control program keeps running after
a fault, which is not a library's decision to make.

**Rule E-20.** An error from the device — a failed write, a decoder capacity
failure — does the same. A terminal that cannot be written to is not a
condition to retry forever.

---

## 9. Open items

- **O-E1 — whether `view` should be allowed to fail.** As specified it returns
  `Result<NIL>` like everything else, and a widget that cannot draw (a
  malformed UTF-8 label) fails the frame. The alternative is that `view` cannot
  fail and bad text renders as U+FFFD. Recommendation: `view` may fail, and the
  *widgets* substitute rather than fail, so the failure path exists but is not
  the ordinary one.
- **O-E2 — multiple `Program`s (screens, modals) in one loop.** Composition is
  currently the application's: it holds its own screen enum and dispatches in
  `update` and `view`. A `Router` helper is plausible and is deferred until
  something written against the library asks for it — the library's own
  examples being the first thing to ask.
