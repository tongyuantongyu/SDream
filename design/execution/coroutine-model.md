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

When EOS arrives, `pull()` returns a sentinel frame where `f.is_eos()` is
true. The framework handles EOS propagation automatically when the coroutine
returns, so a simple passthrough doesn't need explicit EOS logic.

### Example — Transform (Scale)

```cpp
Task scale_filter(FilterContext& ctx) {
    auto [in_fmt, out_fmt] = co_await ctx.negotiate();
    auto scaler = init_scaler(in_fmt, out_fmt);

    while (true) {
        Frame in = co_await ctx.pull(0);
        if (in.is_eos()) break;

        Frame out = co_await ctx.alloc(out_fmt);
        scaler.process(in.buffer(), out.buffer());
        out.copy_timing(in);

        co_await ctx.push(0, std::move(out));
    }
}
```

### Example — Decoder (Multiple Outputs per Input)

Some filters consume one input frame and produce zero or more output frames
(codec delay, B-frame reordering):

```cpp
Task decoder(FilterContext& ctx) {
    auto [in_fmt, out_fmt] = co_await ctx.negotiate();
    auto codec = open_codec(in_fmt, out_fmt);

    while (true) {
        Frame pkt = co_await ctx.pull(0);

        int n = codec.decode(pkt);        // may buffer internally
        while (n-- > 0) {
            Frame decoded = codec.receive();
            Frame out = co_await ctx.alloc(out_fmt);
            copy_to_buffer(decoded, out);
            co_await ctx.push(0, std::move(out));
        }

        if (pkt.is_eos()) {
            // Flush delayed frames
            while (auto flushed = codec.flush()) {
                Frame out = co_await ctx.alloc(out_fmt);
                copy_to_buffer(*flushed, out);
                co_await ctx.push(0, std::move(out));
            }
            break;
        }
    }
}
```

### Example — Multi-Input (Mixer)

```cpp
Task audio_mixer(FilterContext& ctx) {
    auto fmts = co_await ctx.negotiate();
    const int n_inputs = ctx.input_pin_count();

    while (true) {
        Frame out = co_await ctx.alloc(fmts.output(0));
        clear(out.buffer());

        int eos_count = 0;
        for (int i = 0; i < n_inputs; ++i) {
            Frame in = co_await ctx.pull(i);
            if (in.is_eos()) { ++eos_count; continue; }
            mix_into(out.buffer(), in.buffer());
        }

        if (eos_count == n_inputs) break;
        co_await ctx.push(0, std::move(out));
    }
}
```

---

## Async API Surface

`FilterContext` is the filter's handle to the framework. All methods that may
wait are `co_await`-able.

| Method | Suspends when… | Returns |
|--------|---------------|---------|
| `pull(pin_index)` | Input queue empty | `Frame` (or EOS sentinel) |
| `push(pin_index, frame)` | Output queue full | `void` |
| `alloc(media_type [, device])` | No free buffer in pool | `Frame` with writable buffer |
| `make_writable(frame)` | Need to copy and no free buffer | `Frame` with exclusive buffer |
| `negotiate()` | Negotiation in progress | `NegotiatedFormats` (input & output MediaTypes) |
| `flush_received()` | No flush pending | `FlushEvent` |
| `report_error(err)` | Never (non-blocking) | `void` |
| `get_param<T>(key)` | Never | `T` |

### Concurrency Guarantee

The framework never resumes a filter's coroutine concurrently on two threads.
Within the coroutine body, no synchronization is needed — the filter has
exclusive access to its own state between suspension points.

Between two suspension points, the filter runs to completion on one thread
(though not necessarily the *same* thread across different resumptions — see
[Scheduler](scheduler.md)).

---

## C++20 Mapping

The `Task` return type is a coroutine type whose `promise_type` cooperates
with the scheduler:

```cpp
class Task {
public:
    struct promise_type {
        // Scheduler integration
        std::coroutine_handle<> awaiting;   // who to resume when we finish
        Scheduler*              scheduler;

        Task get_return_object();
        std::suspend_always initial_suspend();  // don't start until scheduled
        auto final_suspend() noexcept;          // notify scheduler on completion
        void return_void();
        void unhandled_exception();             // capture and propagate
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
