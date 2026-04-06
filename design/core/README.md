# Core Abstractions

This section defines the data model — the nouns of the framework.

## Relationship Diagram

```
  ┌─────────────────────────── Graph ───────────────────────────┐
  │                                                             │
  │   ┌─ Filter A ─┐          Link           ┌─ Filter B ─┐   │
  │   │            ╔╧══════╗ (rendezvous ╔════╧╗           │   │
  │   │            ║out pin╟──+format)─▶║in pin║           │   │
  │   │            ╚═══════╝           ╚══════╝            │   │
  │   └────────────┘                          └────────────┘   │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
              │                    │
              ▼                    ▼
           Frame              MediaType
         ┌────────┐          ┌─────────┐
         │ buffer ─────▶ Buffer  major │
         │ pts     │     │ device  sub  │
         │ dts     │     │ layout  props│
         │ props   │     └────────┘─────┘
         └────────┘
```

## Sub-Pages

| Document | Contents |
|----------|----------|
| [Graph & Filter](graph-and-filter.md) | Graph (logical vs physical), Filter lifecycle & categories |
| [Pins & Links](pins-and-links.md) | Pin presence modes, fan-out semantics, Link queues |
| [Frame & Buffer](frame-and-buffer.md) | Frame fields, Buffer layout & alignment, ownership rules |
| [Media Type](media-type.md) | MediaType structure, capability constraints, Timestamp |

## Glossary

| Term | Meaning |
|------|---------|
| **Graph** | A validated set of Filters connected by Links. |
| **Filter** | A processing node with Pins and a coroutine body. |
| **Pin** | A typed connection point (input or output) on a Filter. |
| **Link** | A directed edge connecting an output Pin to an input Pin; a rendezvous point with a negotiated format. |
| **Frame** | A timestamped reference to a Buffer plus metadata. |
| **Buffer** | A contiguous or multi-plane block of memory on a specific Device. |
| **MediaType** | A concrete description of the data format flowing through a Link. |
| **Capability** | A set of MediaType constraints that a Pin advertises. |
