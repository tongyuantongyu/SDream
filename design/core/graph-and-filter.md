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
  ┌─────────┐   configure()   ┌────────────┐   prepare()   ┌────────────┐
  │ Created  ├───────────────▶│ Configured  ├─────────────▶│ Negotiated │
  └─────────┘                 └────────────┘               └─────┬──────┘
                                                                 │ run()
                                                                 ▼
                              ┌─────────┐                  ┌──────────┐
                              │  Done   │◀── EOS / return ─│ Running  │
                              └─────────┘                  └────┬─────┘
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
| **Done** | Coroutine returned (normally or via uncaught `EndOfStream`). Resources released. |
| **Error** | Unrecoverable failure. Framework applies the error policy (teardown, isolate, or restart). |

EOS is delivered as an `EndOfStream` exception from `pull()`. Simple filters
need zero EOS handling — the exception propagates, the coroutine terminates,
and the filter transitions to Done. Filters with internal buffering (encoders,
lookahead) catch `EndOfStream` to flush delayed frames before returning. There
is no separate Draining state — flushing is just the filter executing its
`catch (EndOfStream&)` block.

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

