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
  reimplement codecs. Codec filters wrap existing libraries (FFmpeg, hardware
  decoders, etc.). See **[D12]**.
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
