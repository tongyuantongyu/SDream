# Stream Events

## Overview

Beyond frames, the graph carries **events** — control messages that coordinate
state changes across filters. Events fall into three delivery categories:

| Category | Direction | Delivery mechanism |
|----------|-----------|-------------------|
| **In-band** | downstream | Queued with frames; ordering preserved. |
| **Event-exceptions** | to filter | Thrown from the filter's next `co_await`; see [D05+D07](../decisions.md#d05--d07-event-exception-pattern-format-change--error-handling). |
| **Upstream** | sink → source | Out-of-band path (bypasses queues). |

## In-Band Events (Downstream)

These travel in the data direction and are interleaved with frames in the link,
preserving ordering.

| Event | Payload | Meaning |
|-------|---------|---------|
| `EOS` | (none) | No more data on this stream. |
| `Gap` | `{pts, duration}` | No data exists for this time range (silence, still frame). |

## Event-Exceptions

These are delivered by the framework directly to the filter's coroutine. The
next `co_await` call (`pull`, `push`, `alloc`, …) throws the exception instead
of returning normally.

| Exception | Trigger |
|-----------|---------|
| `Flushed` | Seek triggered a pipeline flush. |
| `ParamChanged` | Runtime parameter was modified. |
| `FormatChanged` | Upstream format changed (resolution, pixel format, etc.). |

### Two-Tier Filter Model

| Filter declares | On event | Framework behavior |
|---|---|---|
| `format_resilient` | Exception **not thrown** | Filter processes the next frame with the new format naturally. For filters whose logic is inherently format-agnostic (per-pixel transforms, format normalizers). These filters may have state — the name refers to their format-handling capability, not statelessness. |
| *(default)* | Exception thrown at next `co_await` | If the filter catches the exception → framework assumes handled, continues. If the exception propagates → framework destroys + recreates the filter with new parameters/format. |

### Examples

```cpp
// Default filter — doesn't catch events → auto-recreated on any event
Task simple(FilterContext& ctx) {
    auto fmt = co_await ctx.negotiate();
    while (true) {
        Frame in = co_await ctx.pull(0);
        Frame out = co_await ctx.alloc(fmt);
        process(in, out);
        co_await ctx.push(0, std::move(out));
    }
}

// Filter that handles flush and format change explicitly
Task decoder(FilterContext& ctx) {
    auto fmt = co_await ctx.negotiate();
    auto codec = open_codec(fmt);
    while (true) {
        try {
            Frame pkt = co_await ctx.pull(0);
            // … decode and push …
        } catch (Flushed&) {
            codec.flush();
        } catch (FormatChanged& e) {
            fmt = e.new_format();
            codec = open_codec(fmt);
        }
    }
}

// Filter that cannot handle certain format changes — throws fatal error
Task frame_interp(FilterContext& ctx) {
    auto fmt = co_await ctx.negotiate();
    auto state = init_interp(fmt);
    while (true) {
        try {
            Frame in = co_await ctx.pull(0);
            // … interpolation logic …
        } catch (FormatChanged& e) {
            if (e.new_format().get<int>("width") != fmt.get<int>("width") ||
                e.new_format().get<int>("height") != fmt.get<int>("height"))
                e.as_fatal("interpolation requires consistent resolution");
            fmt = e.new_format();
            state = init_interp(fmt);
        }
    }
}
```

All event-exception types provide an `as_fatal(reason)` convenience method
that re-throws as a `FatalError` with diagnostic context (filter name, event
type, and the caller-supplied reason). The framework handles this the same as
any fatal filter error — applying its error recovery policy (teardown, isolate,
or restart).

This replaces a declarative "format veto" mechanism: the filter itself decides
at runtime whether a given change is survivable, which is more expressive (the
decision can depend on actual values and internal state, not just which
property changed).
```

### Cross-Language Mapping

Each language uses its natural mechanism:

| Language | Event delivery |
|---|---|
| C++ | `throw` / `catch` |
| Python | `raise` / `except` |
| C# | `throw` / `catch` |
| Rust | `pull()` returns `Result<Frame, Event>` — pattern match. The proc-macro auto-generates a default handler that maps unhandled events to a "recreate" signal. |
| C (step fn) | Framework calls `step()` with `response.action = SDREAM_EVENT_*`. Filter returns "handled" or `SDREAM_ACTION_ERROR` to request recreate. |

## Upstream Events (Sink → Source)

These travel against data flow through a separate out-of-band channel — they
must not wait behind queued frames.

| Event | Payload | Meaning |
|-------|---------|---------|
| `SeekRequest` | `{target_ts, flags}` | Request to seek to a position. Propagated from sink toward source. |
| `QoS` | `{jitter, proportion, timestamp}` | Quality-of-service feedback. Tells upstream it is too slow. |
| `Latency` | `{min, max}` | Latency query/report. Each filter adds its own latency and forwards. |

Upstream events are delivered to the filter through a separate mechanism (not
through `pull()`):

```cpp
if (auto qos = co_await ctx.try_receive_upstream_event()) {
    handle_qos(*qos);
}
```

Or a filter can register a callback for upstream events during setup.

---

## EOS Protocol

1. A source finishes producing data. Its coroutine returns (or explicitly
   calls `ctx.signal_eos(pin)`).
2. The framework enqueues an `EOS` event on each output link.
3. Downstream filters receive `EOS` from `pull()` (returned as a sentinel
   frame where `frame.is_eos() == true`).
4. Upon receiving EOS on all input pins, a filter enters the **Draining**
   state: it flushes any internally buffered frames, pushes them downstream,
   then returns from its coroutine (propagating EOS).
5. When all sinks have received EOS, the graph is done.

### Partial EOS

In a multi-input filter (e.g. a muxer), EOS may arrive on some inputs before
others. The filter decides how to handle this — typically it continues
processing the remaining inputs and signals EOS on its output only when all
inputs are done.

---

## Seek Protocol (Flush-Seek)

### Step 1: Request

The application (or a sink) issues a seek:

```cpp
graph.seek(target_timestamp, SeekFlags::Accurate);
```

This sends a `SeekRequest` upstream through the graph.

### Step 2: Source Repositions

The source filter (e.g. file reader + demuxer) receives the seek request and
repositions its read cursor.

### Step 3: Flush via Exception

The framework delivers a `Flushed` exception to all filters in the affected
path:

1. Each suspended filter's next `co_await` throws `Flushed`.
2. **Stateless filters:** exception suppressed, no action needed.
3. **Filters that catch `Flushed`:** discard internal buffers, reset state,
   continue in the main loop.
4. **Filters that don't catch:** exception propagates; framework destroys the
   filter and creates a fresh instance from the same factory.
5. The framework drains any remaining frames from link queues.

### Step 4: Resume

Source begins pushing frames from the new position with absolute timestamps.
Downstream filters (fresh or reset) process normally.

### Seek Flags

| Flag | Meaning |
|------|---------|
| `Accurate` | Seek to the exact requested timestamp (may require decoding from the previous keyframe). |
| `Keyframe` | Seek to the nearest keyframe at or before the requested timestamp. Faster but imprecise. |
| `SnapForward` | Seek to the nearest keyframe at or after the requested timestamp. |
