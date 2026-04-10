# Design Decisions

Resolved decisions are recorded in the [summary](#summary) at the bottom.
Their details have been incorporated into the relevant design documents.

---

## D13. Configuration & Parameter System

| | **Stringly-typed** | **Schema-typed** |
|---|---|---|
| Validation | At filter init (late) | At graph construction (early) |
| Introspection / GUI | Difficult | Straightforward |

> **A:** Schema-typed would be best. But I want auto-generated strongly-typed
> glue code for user when constructing graphs, and the user-facing API also
> needs to support multiple languages. We may need to define parameters using
> protobuf. (Is it too heavy or just what I want?)

**Response:**

Protobuf is a **message format**, not a **parameter schema format**. It lacks
valid ranges, UI hints, dynamic/runtime-changeable flags, and enum labels.
Custom protobuf extensions would be as much work as a custom IDL, plus carry
the protobuf dependency.

**Option A — Custom IDL + codegen tool** (recommended):

```
filter "my.video.scale" {
    input  "in":  video;
    output "out": video;

    param width:     int = 1920 [min=1, max=8192, label="Width", dynamic];
    param algorithm: enum { bilinear, bicubic, lanczos } = lanczos;

    device: cuda, cpu;
}
```

A `sdream-codegen` tool reads this and generates C header, C++ wrapper, Python
class, Rust struct, and JSON Schema.

**Option B — JSON Schema + thin DSL wrapper:**

JSON Schema supports ranges, enums, descriptions. Many languages have existing
codegen tools. Less expressive but zero parser work.

> **A:** Definitely either A or B. Need more time to decide.
>
> The parameter definition should be embedded in the plugin file (DLL), so at
> least on Windows we can have drag-and-drop single-DLL filters.
>
> There will also need to be a reflective way to set parameters, like creating
> a graph from JSON or other structured config files.

**Response:**

Both constraints are already satisfied by the current C ABI design.

**Embedded in DLL:** The `SDreamFilterFactory` struct (registered by the
plugin entry point) contains the full parameter schema inline —
`SDreamParamDesc* params` with types, defaults, ranges, and labels. This is
compiled into the DLL. No external schema files needed. A single
`sdream_plugin_foo.dll` is fully self-describing.

**Reflective access:** The framework loads the DLL, calls the plugin entry
point, and receives factory structs with complete parameter schemas. This
enables:

- **JSON graph construction:** parse parameter values from JSON, validate
  against the factory's schema, report errors with type and range info.
- **CLI:** `--param filter.width=1920` — the framework can validate and
  convert string values using the schema.
- **GUI generation:** iterate parameter descriptors to build property panels.

So the choice between A and B is purely about **developer authoring ergonomics**
— how the filter developer writes the schema. Both produce the same C ABI
factory struct at compile time. The runtime introspection and single-DLL
distribution work identically either way.

| | **Option A (Custom IDL)** | **Option B (JSON Schema)** |
|---|---|---|
| Expressiveness | Full — domain-specific syntax | Good — standard but verbose |
| Parser effort | Small (custom tool) | Zero (existing parsers) |
| Codegen effort | Custom tool for all target languages | Leverage existing JSON Schema codegen |
| Feels native to C++ devs | Yes (DSL next to code) | Less (JSON file next to code) |
| Community familiarity | None (custom) | High (JSON Schema is well-known) |

---

## Summary

### Resolved

| ID | Topic | Resolution | Updated in |
|----|-------|------------|------------|
| D01 | Graph topology | DAG-only (no cycles) | [graph-and-filter.md](core/graph-and-filter.md) |
| D02 | Scheduling strategy | Multiple pluggable schedulers | [scheduler.md](execution/scheduler.md) |
| D03 | Threading model | Device-affine thread groups; simplest first, upgrade later | [scheduler.md](execution/scheduler.md) |
| D04 | Buffer ownership | Shared (ref-counted) + `make_writable` / `try_make_writable` | [frame-and-buffer.md](core/frame-and-buffer.md) |
| D05+D07 | Event-exception pattern | Two-tier: default (throw, catch or recreate) + format-resilient (suppress). Fatal incompatibility handled by filter throwing hard error from catch block. Per-language natural idiom. | [stream-events.md](pipeline/stream-events.md) |
| D06 | Plugin ABI | C ABI + C++ header-only SDK; no C++ ABI | [plugin/](plugin/) |
| D08 | Seek model | Flush-seek; `Flushed` exception | [stream-events.md](pipeline/stream-events.md) |
| D09 | Timestamps | Per-link rational time base, absolute within stream | [media-type.md](core/media-type.md) |
| D10 | Graph dynamism | Parameter-only; `ParamChanged` exception | [graph-api.md](graph-api.md) |
| D11 | Queue sizing | Rendezvous links; BufferQueue and FrameCache as explicit filters | [pins-and-links.md](core/pins-and-links.md) |
| D12 | FFmpeg integration | Hybrid — wrap FFmpeg, native replacements over time | [overview.md](overview.md) |
| D14 | In-place processing | `try_make_writable` / `make_writable` | [frame-and-buffer.md](core/frame-and-buffer.md) |

### Open

| ID | Topic | Status |
|----|-------|--------|
| D13 | Parameter system | Schema-typed; A (custom IDL) or B (JSON Schema) |
