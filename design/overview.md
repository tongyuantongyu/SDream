# Overview

## What SDream Is

SDream is a graph-based multimedia processing framework. Developers write
**filters** — self-contained processing units. Users wire filters into
**graphs** that describe a data-flow pipeline over video, audio, subtitles, and
arbitrary side-data.

Implementation language: **C++20** — native coroutine support, zero-cost
abstractions, direct access to GPU and OS APIs, and the richest existing
multimedia/codec ecosystem.

## Goals

1. **Correctness by construction.** The graph, type, and device systems should
   catch errors at build time rather than at runtime.
2. **Predictable resource usage.** Bounded queues, pooled buffers, and
   backpressure ensure the pipeline does not silently consume unbounded memory.
3. **High throughput and low latency.** The coroutine scheduler should
   efficiently saturate available CPU cores and overlap CPU/GPU work.
4. **Extensibility.** New filter types, device backends, and media types can be
   added without modifying the core.
5. **Cross-language accessibility.** Filter authors should not be forced to
   write C++. A stable C ABI and explicit state-machine protocol make bindings
   straightforward.

## Design Pillars

### Pillar 1 — Coroutine-Based Filters

Every potentially-blocking operation — pulling input, pushing output,
allocating buffers, waiting for device work — is an `co_await`-able call
provided by the framework. The framework owns scheduling and resumes filters
when their dependencies are satisfied.

For non-C++ languages, the same model is expressed as an explicit state machine
behind a C ABI step function. Official scaffolding handles the boilerplate for
C, Rust, Python, and C#.

### Pillar 2 — Heterogeneous-Aware Execution

A graph may mix filters targeting different compute devices (CPU, CUDA, Vulkan,
OpenCL, …). The framework:

- Assigns filters to concrete devices.
- Detects cross-device links and inserts transparent transfer filters.
- Manages device-specific buffer pools.
- Exposes an extensible `DeviceProvider` interface so new backends (Metal,
  SYCL, DirectML, …) can be added as plugins.

## Non-Goals

- **Being a codec library.** SDream orchestrates processing; it does not
  reimplement codecs. Codec filters wrap existing libraries (initially FFmpeg,
  with native replacements added over time).
- **Being a media player.** A player can be built *on top of* SDream, but the
  framework itself is a processing engine, not a UI application.
- **Real-time kernel guarantees.** SDream targets soft real-time (live
  streaming, playback) but does not claim hard real-time determinism.

## Prior Art

| Framework | Relationship to SDream |
|---|---|
| **GStreamer** | Closest in scope (full pipeline framework). SDream replaces GStreamer's thread-per-element + message-bus model with coroutines + scheduler, and adds first-class heterogeneous device support. |
| **libavfilter (FFmpeg)** | Filter graph only; no device abstraction, no async model, limited threading. SDream wraps FFmpeg as a codec backend, not as a framework. |
| **DirectShow / Media Foundation** | Windows-only COM-based frameworks. SDream is portable and uses a simpler C ABI plugin model. |
| **VapourSynth** | Pull-based, frame-server oriented (great for scripting, poor for live/streaming). SDream's coroutine model subsumes pull semantics while also supporting push and live use cases. |

## Comments

> Generic blob hierarchy (coded bitstream, tensors, video, audio, subtitles).
> Framework knows more about special blob types, but at transport level it's
> all blobs. Evaluate impact.

**Response:** The current design already works this way — Frame+Buffer is generic, MediaType is extensible. The impact of making this framing explicit:

- **Core stays type-agnostic.** Transport, scheduling, backpressure, and device migration never inspect buffer contents. They only use MediaType tags for negotiation and pool keying.
- **Ecosystem is type-aware.** Knowledge of video planes, audio channels, tensor shapes lives in standard-library filters and in MediaType standard properties — not in the core.
- **Enables non-media workflows.** AI pipelines (tensor in → inference → tensor out), sensor processing, and generic data pipelines work without core changes.
- **Documentation shift.** Present SDream as a "typed data-flow processing framework" with first-class multimedia support, not a "multimedia framework that also handles other data."

The only design adjustment: BufferLayout is currently video-centric (planes, strides). For tensors, we'd need a `TensorLayout` (shape, dtype, strides per dimension). This can be a separate layout type alongside BufferLayout, or a generalization. Worth considering when defining the Buffer model more precisely.

---

> Multiple schedulers: max utilization, constant speed, blueprint.
> Evaluate impact.

**Response:** These map well to the pluggable scheduler design:

| Scheduler | Strategy | Implementation sketch |
|---|---|---|
| **Max utilization** | Wake filters aggressively, fill all queues, maximize pipeline parallelism. | Hybrid/level-based policy. Multi-threaded. Auto-insert BufferQueues generously. |
| **Constant speed** | Pace source to match sink's real-time rate. Regulate how far ahead the source can get. | Monitor pipeline clock; throttle source filters when output latency exceeds target. Useful for playback. |
| **Blueprint** | Execute filters in a fixed topological order, one frame at a time. Fully deterministic. | Single-threaded. No auto-inserted buffers. Same input always produces same output with same timing. |

Impact on the scheduler interface: the current `select(ready_set)` abstraction already supports all three — only the selection heuristic differs. The "constant speed" scheduler also needs access to the pipeline clock, which is already available. No interface changes needed.

---

> Hard real-time determinism — what are the extra efforts?

**Response:** Going from soft RT to hard RT requires:

1. **Deterministic allocation.** All buffers pre-allocated at `prepare()`. No `malloc` in the hot path. Requires static analysis of pool sizes.
2. **WCET budgets.** Each filter reports worst-case execution time. Scheduler plans within deadline budgets.
3. **Lock-free / wait-free structures.** All scheduler queues and ready-sets must avoid blocking.
4. **Page pinning.** All memory page-locked to prevent OS page faults.
5. **Priority inheritance.** Thread priorities managed to avoid inversion.
6. **No dynamic dispatch in hot path.** Virtual calls and exceptions (even table-based EH) hurt predictability.

Items 1 and 3 are achievable as optional modes. Items 2 and 5 require significant scheduler work. Item 6 conflicts with the event-exception pattern (though the "blueprint" scheduler could use a non-exception delivery path).

Practical approach: the "blueprint" scheduler gets close to hard RT without requiring all of the above — deterministic execution order + pre-allocated pools + single thread covers many professional use cases. True hard RT (broadcast, medical) would need the full list.
