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
| `always` | Pin exists as soon as the filter is created. Must be connected before `prepare()`. | The single input of a scaler. |
| `sometimes` | Pin is created at runtime by the filter when it discovers what it needs. | A demuxer creating output pins per stream. |
| `request` | Pin is created on demand when the user or another filter requests a connection. | Additional audio inputs on a mixer. |

`sometimes` and `request` pins are collectively called **dynamic pins**. The
graph construction phase must wait for dynamic pins to be materialized before
running format negotiation on the affected links.

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
├── media_type     : MediaType         // negotiated concrete format
└── device_info    : DevicePair        // source device, sink device
```

A link is a **rendezvous point** — it has no built-in queue. The producer's
`push()` and the consumer's `pull()` are symmetric suspension points: a frame
passes directly when both sides are ready. If the producer pushes before the
consumer pulls, the producer suspends. If the consumer pulls before the
producer pushes, the consumer suspends. This is the simplest possible link
semantics.

In-band events (`EOS`, `Gap`) are delivered through the same rendezvous,
preserving ordering with frames.

### Buffering as Explicit Filters

Buffering is separated from the link abstraction. Two framework-provided
filters serve this role:

| Filter | Purpose |
|--------|---------|
| **BufferQueue** | Bounded FIFO of configurable capacity. Decouples producer and consumer timing, enabling pipeline parallelism. |
| **FrameCache** | Sliding window of recent frames addressable by index or timestamp. For temporal algorithms (denoising, interpolation, scene detection). |

These are filters in the physical graph — they participate in scheduling,
backpressure, and profiling like any other filter.

### Auto-Inserted Buffering

During `prepare()`, the framework auto-inserts `BufferQueue` filters where
the rendezvous model would otherwise limit throughput or correctness:

| Situation | Reason |
|-----------|--------|
| **Fan-out branches** | Without buffering, `push()` would block until the slowest consumer pulls. A 1-frame buffer on each branch lets the producer continue. |
| **Cross-device links** | Pipeline overlap between CPU and GPU work requires at least 1-frame decoupling. (These links already get a transfer filter; a BufferQueue is inserted alongside it.) |
| **Cross-thread links** | If the scheduler assigns producer and consumer to different threads, a buffer enables them to run in parallel. |

```
Logical:   [decoder] ──────────────────────── [encoder]
Physical:  [decoder] → [buffer(1)] → [upload] → [encoder]
```

The user's logical graph is never cluttered. The physical graph shows buffering
explicitly, visible in debugging and profiling views.

### Fan-Out (One-to-Many)

An output pin may be connected to **multiple** input pins. This creates a
**broadcast**: every frame produced on that output is delivered to all
connected inputs. With reference-counted buffers, this is zero-copy — each
downstream receives a shared reference to the same buffer.

Each fan-out branch has its own independent link (and therefore its own
negotiated format and potentially its own device). The framework auto-inserts a
`BufferQueue` on each branch so that the producer is not blocked by the slowest
consumer.

### Fan-In (Many-to-One)

An input pin accepts exactly **one** upstream link. If a filter needs to
combine multiple streams, it declares multiple input pins (e.g. a mixer has
`input_0`, `input_1`, … or uses `request`-presence pins).

This asymmetry is deliberate: it avoids ambiguity about ordering and ownership
when multiple producers target the same input.

## Questions

> Presence Modes

I'd like to know how this works exactly. Consider these cases:

- General process, with video and audio branch. Audio branch is ignored when input only contains video.

- Fallback, generate silent audio if absent.

- Subtitle embedding, hard fail when input does not contain subtitle.
