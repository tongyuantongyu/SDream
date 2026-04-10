# Graph & Filter

## Graph

### Logical vs Physical Graph

Users construct a **logical graph** — the filters and links they explicitly
specify.

Before execution, the framework transforms it into a **physical graph** by:

1. **Inserting auto-converters** where linked pins disagree on format (e.g. a
   pixel-format converter between NV12 output and I420 input).
2. **Inserting transfer filters** where linked filters reside on different
   devices (see [Data Migration](../heterogeneous/data-migration.md)).
3. **Materializing dynamic pins** (e.g. a demuxer that creates one output pin
   per stream discovered in the container).

The user's logical graph is preserved for inspection. Debugging and profiling
tools can display either view.

### Topology

The graph is a directed acyclic graph (DAG). Cycles are not permitted — all
practical multimedia feedback use cases (echo, feedback effects, iterative
refinement) are modeled as intra-filter behavior. The acyclic guarantee means
a topological ordering always exists, which simplifies scheduling and
eliminates a class of deadlocks.

### Graph Connectivity

A Graph must be a single connected component. If a user needs independent
pipelines (e.g. processing multiple files concurrently), they create separate
Graph instances. The Scheduler can manage multiple Graphs, scheduling them
independently — each can be started, flushed, and torn down in isolation.

---

## Filter

### Lifecycle

```
  ┌─────────┐     configure()     ┌────────────┐    prepare()    ┌────────────┐
  │ Created  ├───────────────────▶│ Configured  ├──────────────▶│ Negotiated │
  └─────────┘                     └────────────┘                └─────┬──────┘
                                                                      │ run()
                                                                      ▼
  ┌─────────┐   EOS propagated    ┌────────────┐                ┌──────────┐
  │  Done   │◀────────────────────┤  Draining  │◀── EOS in ────│ Running  │
  └─────────┘                     └────────────┘                └────┬─────┘
                                                                     │
                                        ┌─────────┐                  │
                                        │  Error  │◀── fatal ────────┘
                                        └─────────┘
```

| State | Description |
|-------|-------------|
| **Created** | Instance allocated, identity assigned. Parameters may be set. |
| **Configured** | Pins declared, static capabilities published. No data flows yet. |
| **Negotiated** | Format negotiation complete; concrete MediaTypes fixed on all links. |
| **Running** | Coroutine body is live. Filter pulls, processes, pushes. |
| **Draining** | Upstream delivered EOS. Filter flushes internal buffered data (e.g. encoder delay frames), then signals EOS downstream. |
| **Done** | All resources released. Filter can be destroyed. |
| **Error** | Unrecoverable failure. Framework applies the error policy (teardown, isolate, or restart). |

### Thread-Safety Contract

The framework guarantees that a filter's coroutine body (or step function) is
**never called concurrently from multiple threads**. A filter's internal state
does not need synchronization. Shared data between filters flows exclusively
through the Link queues and Frame references.

### Filter Categories

While every filter uses the same coroutine interface, categorizing filters by
their pin topology helps the scheduler and tooling:

| Category | Input pins | Output pins | Examples |
|----------|-----------|-------------|----------|
| **Source** | 0 | ≥1 | File reader, capture device, test pattern generator |
| **Sink** | ≥1 | 0 | File writer, display output, null sink |
| **Transform** | ≥1 | ≥1 | Scaler, color converter, resampler, encoder, decoder |
| **Demuxer** | 1 | dynamic | Container demuxer (creates pins per discovered stream) |
| **Muxer** | dynamic | 1 | Container muxer (accepts pins per stream to interleave) |

These categories are advisory metadata on the filter factory — the coroutine
model is uniform.

### Filter Instances vs Factories

A **FilterFactory** is a static descriptor registered by a plugin. It
describes the filter's name, category, pin templates, supported devices, and
parameter schema.

A **Filter instance** is created from a factory at graph-construction time. The
same factory can produce multiple independent instances (e.g. two separate
`scale` filters with different parameters in the same graph).

## Comments

> Dynamic pins complexity. Dig into this topic further.

**Response:** Two approaches:

**A — Probe-then-link (simpler):** The demuxer has a `probe()` method called before graph construction. It reads the container header and creates output pins based on what it finds. The user then links these known pins. All pin creation happens before `prepare()`. This is what FFmpeg's `avformat_find_stream_info()` does.

**B — Deferred materialization (current design):** Pins are created during `prepare()`. The framework pauses, notifies the application that new pins appeared, waits for them to be connected, then resumes negotiation.

Approach A is simpler and covers 95% of cases (file-based sources know their streams upfront). Approach B is needed for live sources where streams appear/disappear dynamically (e.g. MPEG-TS with changing program tables).

Recommendation: start with A as the standard path. B can be added later for live use cases, as a `prepare()`-time callback: `on_new_pin(pin) → return connection target or null`.

---

> EOS as exception. Simple filters need zero extra code.

**Response:** Endorsed — this fits the event-exception pattern perfectly.

```cpp
// Simple filter: EOS propagates → coroutine terminates → Done. Zero extra code.
Task passthrough(FilterContext& ctx) {
    while (true) {
        Frame f = co_await ctx.pull(0);  // throws EndOfStream
        co_await ctx.push(0, std::move(f));
    }
}

// Decoder: catches EOS to flush delayed frames
Task decoder(FilterContext& ctx) {
    auto codec = open_codec(co_await ctx.negotiate());
    try {
        while (true) {
            Frame pkt = co_await ctx.pull(0);
            // decode and push...
        }
    } catch (EndOfStream&) {
        while (auto f = codec.flush())
            co_await ctx.push(0, *f);
    }
}
```

Key clarification: **uncaught `EndOfStream` = normal termination (→ Done state), NOT recreate.** This differs from `FormatChanged`/`Flushed` where uncaught = recreate. The framework distinguishes exception types in `unhandled_exception()`.

The Draining lifecycle state becomes implicit — it's "the filter is executing its `catch (EndOfStream&)` block."

If you confirm, I'll update the lifecycle diagram, stream-events, and coroutine-model examples.

---

> FilterFactory per-device? Shared per-device resources?

**Response:** Factory should stay **global** (one factory per filter type). Reasons:

- A factory is metadata (name, pins, params, device affinity list). It has no device-specific state.
- The same H.264 decoder factory produces instances for CPU, CUDA, or VAAPI — the instance adapts to the assigned device, not the factory.

For **shared per-device resources** (e.g. hardware decoder sessions, pre-allocated scratch buffers shared among instances on the same GPU): this is the DeviceProvider's responsibility. The provider manages per-device state, and filter instances request shared resources from it:

```cpp
auto session = ctx.device_provider().get_shared<HwDecoderSession>(device);
```

This keeps the factory simple (static descriptor) and centralizes device resource management in the provider (which already owns the device context). Not worth building a separate framework mechanism — let the provider handle it.
