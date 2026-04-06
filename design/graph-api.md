# Graph Construction API

## Programmatic API (C++)

```cpp
Graph g;

// Add filters — returns a FilterHandle for linking
auto src    = g.add("file_source",     {{"path", "/input.mp4"}});
auto demux  = g.add("mp4_demux");
auto decode = g.add("h264_decode");
auto scale  = g.add("scale",          {{"width", 1920}, {"height", 1080}});
auto encode = g.add("h264_encode",    {{"bitrate", 8'000'000}});
auto mux    = g.add("mp4_mux");
auto sink   = g.add("file_sink",      {{"path", "/output.mp4"}});

// Link by pin name
g.link(src/"out",       demux/"in");
g.link(demux/"video",   decode/"in");
g.link(decode/"out",    scale/"in");
g.link(scale/"out",     encode/"in");
g.link(encode/"out",    mux/"video");
g.link(mux/"out",       sink/"in");

// Optional: constrain a link's format
g.link(decode/"out", scale/"in", {{"pixel_format", "NV12"}});

// Optional: pin a filter to a device
encode.set_device(Device{"CUDA", 0});

// Prepare: negotiate, insert transfers/converters, validate
g.prepare();

// Run: start scheduler, block until done or error
Status status = g.run();
```

### Runtime Parameter Changes

Graph topology is fixed after `prepare()`. Filter parameters are adjustable
at runtime:

```cpp
// While graph is running (from another thread or a callback):
scale.set_param("width", 1280);
scale.set_param("height", 720);
```

The framework delivers a `ParamChanged` exception to the filter at its next
suspension point. The filter can catch it to re-read parameters and adjust,
or let it propagate (triggering destroy + recreate with the new parameter
values). Stateless filters that declared `dynamic` parameters receive the
new values transparently without interruption.

Whether a parameter change also triggers format renegotiation depends on the
filter — see [Mid-Stream Renegotiation](pipeline/format-negotiation.md#mid-stream-renegotiation).

---

## Declarative Graph Format

A JSON schema for describing graphs without code:

```json
{
  "version": 1,
  "filters": [
    {
      "id": "src",
      "type": "file_source",
      "params": { "path": "/input.mp4" }
    },
    {
      "id": "demux",
      "type": "mp4_demux"
    },
    {
      "id": "decode",
      "type": "h264_decode"
    },
    {
      "id": "scale",
      "type": "scale",
      "params": { "width": 1920, "height": 1080 },
      "device": { "provider": "CUDA", "index": 0 }
    }
  ],
  "links": [
    { "from": "src:out",     "to": "demux:in" },
    { "from": "demux:video", "to": "decode:in" },
    { "from": "decode:out",  "to": "scale:in", "format": { "pixel_format": "NV12" } }
  ],
  "options": {
    "clock": "freerun",
    "low_latency": false
  }
}
```

Use cases:

- Save / load pipeline configurations.
- GUI graph editors export this format.
- Remote pipeline construction (send JSON over a control channel).
- Scripting and automation.

---

## C API

The graph API is also available as C functions for FFI:

```c
SDreamGraph* g = sdream_graph_create();
SDreamFilterHandle src = sdream_graph_add_filter(g, "file_source", params, n_params);
SDreamFilterHandle dec = sdream_graph_add_filter(g, "h264_decode", NULL, 0);
sdream_graph_link(g, src, "out", dec, "in", NULL);
int32_t status = sdream_graph_prepare(g);
if (status == SDREAM_STATUS_OK)
    status = sdream_graph_run(g);
sdream_graph_destroy(g);
```

---

## Validation

`g.prepare()` performs these checks in order:

| Stage | Check | On failure |
|-------|-------|-----------|
| **Topology** | All `always`-presence pins connected. No isolated filters (except if explicitly allowed). | Error: "Filter 'scale' input pin 'in' is not connected." |
| **Cycle** | No cycles (DAG only). | Error: "Cycle detected: A → B → C → A." |
| **Device** | All assigned devices are available. Transfer paths exist for every cross-device link. | Error: "Filter 'encode' requires CUDA but no CUDA device is available." |
| **Negotiation** | Format negotiation converges for every link. | Error: "Cannot negotiate format between 'decode:out' (NV12) and 'filter:in' (RGB24): no auto-converter." |
| **Parameters** | All required parameters set. Values within declared ranges. | Error: "Filter 'scale' parameter 'width' must be in range [1, 8192], got 0." |
| **Resource** | Buffer pool capacity can be satisfied (total GPU memory estimate). | Warning: "Estimated GPU memory usage (4.2 GB) exceeds available (4.0 GB). Pipeline may stall." |

Errors are collected (not fail-fast) so the user sees all problems in one
pass.
