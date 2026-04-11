# Coroutine Model

## Filter Body

A filter's processing logic lives in a single coroutine function. Every I/O
or resource-acquisition call is `co_await`-able — the filter suspends, the
scheduler resumes it when the dependency is satisfied.

### Minimal Example — Passthrough

```cpp
Task passthrough(FilterContext& ctx) {
    while (true) {
        Frame f = co_await ctx.pull(0);
        co_await ctx.push(0, std::move(f));
    }
}
```

No EOS handling needed — `EndOfStream` propagates as an exception, the
coroutine terminates, and the framework transitions the filter to Done.

### Example — Transform (Scale)

```cpp
Task scale_filter(FilterContext& ctx) {
    auto [in_fmt, out_fmt] = co_await ctx.negotiate();
    auto scaler = init_scaler(in_fmt, out_fmt);

    while (true) {
        Frame in = co_await ctx.pull(0);
        Frame out = co_await ctx.alloc_like(in, out_fmt);
        scaler.process(in.buffer(), out.buffer());
        co_await ctx.push(0, std::move(out));
    }
}
```

`alloc_like(src, fmt)` allocates a buffer matching `fmt` and copies timing
(pts, dts, duration) and properties from `src`.

### Example — Decoder (Multiple Outputs per Input)

Some filters consume one input frame and produce zero or more output frames
(codec delay, B-frame reordering):

```cpp
Task decoder(FilterContext& ctx) {
    auto [in_fmt, out_fmt] = co_await ctx.negotiate();
    auto codec = open_codec(in_fmt, out_fmt);

    try {
        while (true) {
            Frame pkt = co_await ctx.pull(0);

            int n = codec.decode(pkt);
            while (n-- > 0) {
                Frame out = co_await ctx.alloc(out_fmt);
                codec.receive_into(out);
                co_await ctx.push(0, std::move(out));
            }
        }
    } catch (EndOfStream&) {
        while (auto flushed = codec.flush()) {
            Frame out = co_await ctx.alloc(out_fmt);
            copy_to_buffer(*flushed, out);
            co_await ctx.push(0, std::move(out));
        }
    }
}
```

### Example — Multi-Input (Mixer)

```cpp
Task audio_mixer(FilterContext& ctx) {
    auto fmts = co_await ctx.negotiate();
    const int n_inputs = ctx.input_pin_count();
    std::vector<bool> eos(n_inputs, false);

    while (!std::all_of(eos.begin(), eos.end(), std::identity{})) {
        Frame out = co_await ctx.alloc(fmts.output(0));
        clear(out.buffer());

        for (int i = 0; i < n_inputs; ++i) {
            if (eos[i]) continue;
            auto result = co_await ctx.try_pull(i);
            if (result.is<EndOfStream>()) { eos[i] = true; continue; }
            mix_into(out.buffer(), result.frame().buffer());
        }

        co_await ctx.push(0, std::move(out));
    }
}
```

`try_pull(pin)` returns `Result<Frame, Event>` instead of throwing — better
for multi-input filters that handle events per-pin.

---

## Async API Surface

`FilterContext` is the filter's handle to the framework. All methods that may
wait are `co_await`-able.

| Method | Suspends when… | Returns |
|--------|---------------|---------|
| `pull(pin)` | Upstream not ready | `Frame`. Throws `EndOfStream`, `Flushed`, `FormatChanged`, `ParamChanged`. |
| `try_pull(pin)` | Upstream not ready | `Result<Frame, Event>`. Never throws — events returned as the `Event` variant. |
| `push(pin, frame)` | Downstream not ready | `void` |
| `alloc(media_type [, device])` | No free buffer in pool | `Frame` with writable buffer |
| `alloc_like(src, media_type)` | No free buffer in pool | `Frame` with writable buffer, timing and properties copied from `src` |
| `make_writable(frame)` | Need to copy and no free buffer | `Frame` with exclusive buffer |
| `try_make_writable(frame)` | Never | `Frame*` — sole owner returns frame, shared returns nullptr |
| `alloc_workspace(size [, mode] [, device])` | No free buffer in pool | `Workspace` handle. Mode: `Unlocked` (default) or `Locked`. See [Workspace Allocation](../heterogeneous/allocation.md#workspace-allocation). |
| `negotiate()` | Negotiation in progress | `NegotiatedFormats` (input & output MediaTypes) |
| `on_device(dev)` | Immediately (migrates thread) | `void` — resumes on the target device's thread |
| `pin_thread()` | Immediately | `ScopeGuard` — scheduler won't migrate until guard is dropped |
| `device_sync(stream)` | Until GPU stream/fence completes | `void` — CPU thread freed for other coroutines |
| `start(awaitable)` | Never (non-blocking) | `Future<T>` — scheduler begins tracking the operation independently (see [Async Combinators](#async-combinators)) |
| `report_error(err)` | Never | `void` |
| `get_param<T>(key)` | Never | `T` |

### Concurrency Guarantee

The framework never resumes a filter's coroutine concurrently on two threads.
Within the coroutine body, no synchronization is needed — the filter has
exclusive access to its own state between suspension points.

Between two suspension points, the filter runs to completion on one thread
(though not necessarily the *same* thread across different resumptions — see
[Scheduler](scheduler.md)). Use `pin_thread()` when same-thread is required.

---

## Async Combinators

### Pending Operations (Futures)

Any async API call can be **started** without suspending the coroutine.
`ctx.start(awaitable)` registers the operation with the scheduler and returns
a `Future<T>` handle. The scheduler fulfills the operation independently;
the filter collects results later.

```cpp
auto pull_f  = ctx.start(ctx.pull(0));         // Future<Frame>
auto push_f  = ctx.start(ctx.push(0, frame));  // Future<void>
```

A `Future` lives until it completes or is explicitly cancelled. There is no
implicit cancellation — pending operations keep running across `wait_any`
calls, unlike one-shot combinators that tear down unfinished work.

### wait_any / wait_all

| Combinator | Behavior |
|------------|----------|
| `wait_any(futures...)` | Suspend until **any** future has completed. Returns its index + result. Other futures **keep running**. |
| `wait_all(futures...)` | Suspend until **all** futures have completed. Returns all results. |

Both accept variadic arguments or a range of futures. If a future has already
completed when the call is made, `wait_any` returns immediately.

The concurrency guarantee is preserved: the filter suspends once, the
scheduler fulfills operations, and the filter resumes once with the result.
No user code runs concurrently.

### Example — Fan-Out Filter

With `buffer_size=1`, the fan-out starts all pushes and waits for all:

```cpp
Frame f = co_await ctx.pull(0);
auto pushes = ctx.start_all(n, [&](int i) { return ctx.push(i, f); });
co_await wait_all(pushes);
```

With `buffer_size>1`, a reactor loop overlaps pulling and pushing. Completed
futures are re-armed with the next operation:

```cpp
Ring ring(buffer_size);
auto pull_f = ctx.start(ctx.pull(0));
std::vector<Future<void>> push_fs(n);  // initially empty

while (true) {
    auto ev = co_await wait_any(pull_f, push_fs...);

    if (ev.is(pull_f)) {
        ring.push_back(ev.result());
        for (int i = 0; i < n; ++i)          // start pushes for new frame
            if (!push_fs[i] && ring.pending(i))
                push_fs[i] = ctx.start(ctx.push(i, ring.front(i)));
        if (ring.has_space())
            pull_f = ctx.start(ctx.pull(0));  // re-arm pull
    }

    if (ev.is_push(i)) {
        ring.advance(i);
        push_fs[i].reset();
        if (ring.pending(i))                  // re-arm push for next frame
            push_fs[i] = ctx.start(ctx.push(i, ring.front(i)));
        if (!pull_f && ring.has_space())
            pull_f = ctx.start(ctx.pull(0));
    }
}
```

The scheduler controls `buffer_size` as a single tuning knob (see
[Scheduler — Fan-Out Buffering](scheduler.md#fan-out-buffering)).

### Example — Priority Multiplexer

```cpp
Task priority_mux(FilterContext& ctx) {
    const int n = ctx.input_pin_count();
    std::vector<Future<Frame>> pulls(n);
    for (int i = 0; i < n; ++i)
        pulls[i] = ctx.start(ctx.pull(i));

    while (true) {
        auto [pin, frame] = co_await wait_any(pulls);
        co_await ctx.push(0, std::move(frame));
        pulls[pin] = ctx.start(ctx.pull(pin));   // re-arm completed pull
    }
}
```

No cancellation, no re-registration — each pull future persists until
fulfilled, then is re-armed for the next frame.

---

## C++20 Mapping

The `Task` return type is a coroutine type whose `promise_type` cooperates
with the scheduler:

```cpp
class Task {
public:
    struct promise_type {
        Scheduler*              scheduler;

        Task get_return_object();
        std::suspend_always initial_suspend();  // don't start until scheduled
        auto final_suspend() noexcept;          // notify scheduler on completion
        void return_void();
        void unhandled_exception();             // EndOfStream → Done; others → recreate or error
    };
    // …
};
```

Each `co_await ctx.pull(…)` (and the other API calls) returns an awaitable
whose `await_suspend` registers the filter's need with the scheduler and
returns `std::noop_coroutine()` (returning control to the scheduler loop,
rather than to a specific other coroutine).

The scheduler, not the coroutine, decides who runs next. This decoupling is
what allows pluggable scheduling policies and device-affine dispatch.
