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

In-band events (`EOS`, `Gap`) are delivered through the same rendezvous,
preserving ordering with frames.

### Buffering as Explicit Filters

Buffering is separated from the link abstraction. Two framework-provided
filter types serve this role:

| Filter | Purpose |
|--------|---------|
| **BufferQueue** | Bounded FIFO of configurable capacity. Decouples producer and consumer timing. |
| **FrameCache** | Sliding window of recent frames addressable by index or timestamp. For temporal algorithms (denoising, interpolation, scene detection). |

These are filters in the physical graph — they participate in scheduling,
backpressure, and profiling like any other filter. The scheduler decides
when and where to auto-insert BufferQueue filters for pipeline parallelism
(see [Scheduler — Auto-Inserted Buffering](../execution/scheduler.md#auto-inserted-buffering)).

### Fan-Out (One-to-Many)

An output pin may be connected to **multiple** input pins. This creates a
**broadcast**: every frame produced on that output is delivered to all
connected inputs. With reference-counted buffers, this is zero-copy — each
downstream receives a shared reference to the same buffer.

Each fan-out branch has its own independent link. All branches share the same
negotiated format and device at the output pin. If a downstream consumer needs
a different format or device, auto-inserted converter/transfer filters bridge
the gap on that specific branch.

### Fan-In (Many-to-One)

An input pin accepts exactly **one** upstream link. If a filter needs to
combine multiple streams, it declares multiple input pins (e.g. a mixer has
`input_0`, `input_1`, … or uses `request`-presence pins).

This asymmetry is deliberate: it avoids ambiguity about ordering and ownership
when multiple producers target the same input.

## Comments

> Presence Modes — how does it work for optional/fallback/required patterns?

**Response:** With concrete examples:

**Optional audio (ignore if absent):** Demuxer's audio output pin has presence=`sometimes`. If the container has no audio, the pin is never created. Downstream audio filters are never linked. The graph runs video-only. The framework validates that `sometimes` pins don't require connection — unconnected `sometimes` pins are fine.

**Fallback (generate silent audio):** This is application-level logic during graph construction, not a framework feature. The graph builder inspects the demuxer's pins after probe. If no audio pin exists, it inserts a silence-generator source. The framework doesn't need to know about fallback semantics.

**Required subtitle (hard fail):** The subtitle embedder declares its subtitle input pin as presence=`always`. During `prepare()`, if that pin is unconnected, validation fails: *"Filter 'subtitle_embed' input pin 'subs' is not connected."* The framework enforces that `always` pins must be connected.

**Other use case — variable-count inputs:** A mixer uses `request`-presence input pins. The user connects 2, 5, or 20 inputs. The mixer's coroutine iterates over however many pins exist. No fixed pin count.

This covers the three presence modes without additional mechanisms. The framework's role is just enforcing `always` = must connect, `sometimes` = may connect, `request` = created on demand.

---

> `Gap` — why needed if VFR timestamps exist?

**Response:** Gap is semantically different from a timestamp jump:

- VFR timestamps: "next frame is at T+500ms" — consumer doesn't know if there's data between now and T+500ms. Could be a stall, could be intentional silence.
- Gap event: "I explicitly have NO data for this time range" — consumer can act on this (render silence, hold last frame, mux filler).

This matters for: audio renderers (must output silence, not just wait), live muxers (must insert filler to maintain bitrate), and subtitle renderers (must clear display).

That said, Gap is a convenience, not core infrastructure. It can be deferred — many pipelines work fine without it. If you'd prefer to leave it out initially and add it when a concrete use case demands it, that's reasonable.

---

> BufferQueue vs FrameCache — should they be separate?

**Response:** They differ in access pattern, not just storage:

- **BufferQueue:** FIFO. Frames consumed (removed) on pull. Producer and consumer advance independently.
- **FrameCache:** Random access. Frames are NOT consumed — they stay cached until evicted by the sliding window. Consumer can `peek(index)` or `peek(timestamp)`.

A unified approach: **FrameCache is a superset.** A FrameCache used in FIFO-only mode (always read front, evict on advance) IS a BufferQueue. If the implementation difference is small, a single filter with a mode flag works. If random access adds meaningful overhead to the FIFO path (index maintenance, eviction policy), keeping them separate is justified.

Recommendation: start with one implementation (FrameCache) and expose both interfaces. Split later if profiling shows the FIFO-only path needs to be leaner.

---
