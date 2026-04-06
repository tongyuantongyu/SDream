# Metadata & Side Data

## Frame Properties (PropertyBag)

Every frame carries a `PropertyBag` — an extensible key-value map for
per-frame metadata that does not fit into the fixed `Frame` fields.

### Property Value Types

| Type | Examples |
|------|---------|
| `int64` | QP value, crop offsets |
| `double` | Confidence score |
| `string` | Scene description |
| `rational` | Display aspect ratio override |
| `blob` | Opaque binary data (motion vectors, SEI payloads) |
| `buffer_ref` | Reference to another Buffer (e.g. thumbnail) |

### Standard Property Keys

The framework defines a namespace of well-known keys. Filters should use
these rather than inventing ad-hoc keys for the same data.

| Key | Type | Meaning |
|-----|------|---------|
| `"crop.left"`, `"crop.top"`, `"crop.right"`, `"crop.bottom"` | int64 | Active region crop offsets (pixels) |
| `"rotation"` | int64 | Display rotation in degrees (0, 90, 180, 270) |
| `"hdr.mastering_display"` | blob | SMPTE ST 2086 mastering display metadata |
| `"hdr.content_light"` | blob | MaxCLL / MaxFALL |
| `"hdr.dynamic"` | blob | HDR10+ or Dolby Vision RPU |
| `"motion_vectors"` | blob | Per-macroblock motion vector table |
| `"regions_of_interest"` | blob | Array of ROI rectangles with weights |
| `"qp_table"` | blob | Per-macroblock QP values |
| `"closed_captions"` | blob | CEA-608/708 CC data |
| `"icc_profile"` | blob | ICC color profile |

Third-party keys should use a reverse-domain prefix (e.g.
`"com.example.my_metadata"`) to avoid collisions.

### Propagation Rules

By default, properties are **forwarded** from input to output when a filter
does not explicitly handle them. This ensures metadata like HDR info or crop
offsets survives a processing chain without every filter needing to know about
them.

A filter can:

- **Read** a property: `auto val = frame.props().get<int64>("rotation");`
- **Set / overwrite** a property: `frame.props().set("rotation", 0);`
- **Remove** a property: `frame.props().remove("crop.left");`
- **Suppress forwarding**: mark specific keys as consumed so they are not
  automatically copied to the output frame.

The `copy_props(src_frame)` helper copies all properties from a source frame
to the current frame, respecting suppressions.

---

## Side-Data Streams

Data that is not temporally aligned 1:1 with video or audio frames — such as
subtitles, chapter markers, timed metadata, or sensor telemetry — flows as
a separate **side-data stream** through its own pins and links.

### Why Not Just Properties?

Properties are per-frame and tied to a specific video/audio frame's lifetime.
Side data has its own timing:

- A subtitle event spans a time range (start PTS → end PTS) that may cover
  many video frames.
- Chapter markers are sparse — one every few minutes.
- Sensor telemetry may arrive at a different rate than the video.

A dedicated stream gives side data its own buffering, backpressure, format
negotiation, and device placement — all the same infrastructure as video/audio.

### Side-Data MediaType

```
MediaType {
    major: "side",
    sub:   "ASS_" / "SRT_" / "WVTT" / "JSON" / …,
    properties: {
        language: "en",
        // codec-specific keys
    }
}
```

### Convergence

When a renderer or muxer needs to combine video + side data, it has separate
input pins for each stream and aligns them by PTS internally.

---

## Custom Major Types

The MediaType system is open — third-party code can introduce new major types
for domain-specific data:

| Major | Use case |
|-------|----------|
| `"sens"` | IMU / accelerometer / gyroscope data |
| `"ctrl"` | Machine-control commands (PTZ, tally) |
| `"dpth"` | Depth maps (structured light, ToF) |
| `"pntc"` | Point cloud data |

The framework treats unknown majors generically — they flow through pins and
links, participate in negotiation, and are buffered/scheduled like any other
stream. Only filters that understand the major type interpret the data.
