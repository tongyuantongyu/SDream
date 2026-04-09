# Pins & Links

## Pins

A pin is a typed connection point on a filter. It has a direction, a
capability set, and a presence mode.

### Pin Properties

| Property | Description |
|----------|-------------|
| **Direction** | `input` or `output`. |
| **Name** | Unique within the filter (e.g. `"video"`, `"audio_0"`). Used in the linking API. |
| **Capability set** | Set of `MediaType` constraints this pin can accept (input) or produce (output). See [Media Type](media-type.md). |
| **Presence** | When this pin exists — see below. |

### Presence Modes

| Mode | Meaning | Example |
|------|---------|---------|
| `always` | Must be connected before `prepare()`. | The single input of a scaler. |
| `optional` | May be left unconnected. Filter handles the absence. | A muxer's subtitle input — works without it. |
| `request` | Created on demand at graph construction time. Pin count is a parameter. | A mixer: user requests 2, 5, or 20 input pins. |

All pins are resolved before `prepare()`. The graph topology never changes
during execution. If a connected pin will carry no data (e.g. a demuxer output
for a stream absent in the container), the filter sends `Absence` — a subtype
of `EndOfStream` — immediately on that pin. Filters that catch `EndOfStream`
work unchanged. Filters that need to distinguish "never had data" from "stream
ended" can catch `Absence` specifically (e.g. a subtitle embedder calls
`absence.as_fatal("subtitle stream required")`).

### Capability Set

A pin's capability set is a list of `MediaType` templates — partially
specified MediaTypes with wildcards or ranges:

```
output_caps = [
    { major: "vide", sub: "NV12", width: [1, 8192], height: [1, 8192] },
    { major: "vide", sub: "I420", width: [1, 8192], height: [1, 8192] },
]
```

During negotiation, the framework intersects the upstream output caps with the
downstream input caps to find a compatible concrete MediaType.

---

## Links

A link connects exactly one output pin to exactly one input pin. It is the
conduit for data between two filters.

### Link Structure

```
Link
├── source_pin     : PinRef            // output pin of upstream filter
├── sink_pin       : PinRef            // input pin of downstream filter
└── media_type     : MediaType         // negotiated concrete format
```

A link always connects two pins that agree on a single format and reside on
the same device. If format conversion or device migration is needed, the
framework auto-inserts converter/transfer filters — the link itself stays
simple.

A link is a **rendezvous point** — it has no built-in queue. The producer's
`push()` and the consumer's `pull()` are symmetric suspension points: a frame
passes directly when both sides are ready. If the producer pushes before the
consumer pulls, the producer suspends. If the consumer pulls before the
producer pushes, the consumer suspends. This is the simplest possible link
semantics.

EOS and other event-exceptions are delivered by the framework at the filter's
next `co_await`, not through the rendezvous. See
[Stream Events](../pipeline/stream-events.md).

### Buffering as Explicit Filters

Buffering is separated from the link abstraction. A framework-provided
**FrameCache** filter serves this role:

- In **FIFO mode**: bounded queue, frames consumed on pull. Decouples producer
  and consumer timing for pipeline parallelism.
- In **cache mode**: sliding window, frames addressable by index or timestamp.
  For temporal algorithms (denoising, interpolation, scene detection). Frames
  are NOT consumed — they stay cached until evicted.

FrameCache is a filter in the physical graph — it participates in scheduling,
backpressure, and profiling like any other filter. The scheduler decides when
and where to auto-insert FrameCache (FIFO mode) for pipeline parallelism
(see [Scheduler — Auto-Inserted Buffering](../execution/scheduler.md#auto-inserted-buffering)).

### Fan-Out (One-to-Many)

A link is always **1:1** — one output pin to one input pin. When the user
connects one output to multiple inputs in the graph API, the framework
auto-inserts a **fan-out filter** during `prepare()`:

```
User view:    A.out ──→ B.in
                    └──→ C.in

Physical:     A.out ──→ FanOut.in
                        FanOut.out_0 ──→ B.in
                        FanOut.out_1 ──→ C.in
```

The fan-out filter has a scheduler-controlled `buffer_size`. With
`buffer_size=1`, it broadcasts each frame via `when_all`. With larger sizes,
it maintains an internal ring buffer with per-branch read cursors, allowing
branches to advance independently (see
[Async Combinators](../execution/coroutine-model.md#async-combinators)).
With reference-counted buffers, delivery is zero-copy — each downstream
receives a shared reference to the same buffer.

All branches share the same negotiated format and device at the fan-out's
input pin. If a downstream consumer needs a different format or device,
auto-inserted converter/transfer filters bridge the gap on that specific
branch.

### Fan-In (Many-to-One)

An input pin accepts exactly **one** upstream link. If a filter needs to
combine multiple streams, it declares multiple input pins (e.g. a mixer has
`input_0`, `input_1`, … or uses `request`-presence pins).

This asymmetry is deliberate: it avoids ambiguity about ordering and ownership
when multiple producers target the same input.

