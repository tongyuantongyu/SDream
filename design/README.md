# SDream Design Documentation

SDream is a graph-based multimedia processing framework built on two pillars:
**coroutine-based filters** and **heterogeneous-aware execution**.

## Reading Order

| # | Document | What it covers |
|---|----------|----------------|
| 1 | [Overview](overview.md) | Goals, non-goals, design principles, prior art |
| 2 | [Core Abstractions](core/) | Graph, Filter, Pin, Link, Frame, Buffer, MediaType |
| 3 | [Execution Model](execution/) | Coroutine filter body, async API, scheduler |
| 4 | [Heterogeneous Computing](heterogeneous/) | Device abstraction, memory model, auto-migration |
| 5 | [Pipeline Mechanics](pipeline/) | Format negotiation, stream events, seeking, errors |
| 6 | [Timing & Synchronization](timing-and-sync.md) | Clock, A/V sync, QoS, latency |
| 7 | [Plugin System](plugin/) | C ABI, filter factory, cross-language scaffolding |
| 8 | [Graph API](graph-api.md) | Programmatic & declarative construction, validation |
| 9 | [Metadata](metadata.md) | Frame properties, side-data streams |
| 10 | [Observability](observability.md) | Logging, graph visualization, profiling |
| 11 | [Design Decisions](decisions.md) | Trade-offs requiring resolution |

## Notation

- Struct layouts use a tree notation: `├──` for fields, types after `:`.
- C++ snippets are illustrative of intent, not final API.
- C ABI snippets are closer to final — the plugin boundary is precisely
  specified.
- **[D13]** references point to the one remaining open entry in the
  [decisions](decisions.md) document.
