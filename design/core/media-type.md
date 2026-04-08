# Media Type & Timestamp

## MediaType

A MediaType is the concrete description of data format on a link. It combines
a structured type tag with an extensible property map.

```
MediaType
├── major          : TypeID             // category: video, audio, side, tensor, …
├── sub            : TypeID             // subtype: NV12, I420, f32, ASS, …
└── properties     : PropertyMap        // namespaced key → Value
```

`TypeID` is a structured 64-bit integer. Bits [63:48] encode the category
(framework-defined). Bits [47:0] encode the subtype (framework-assigned for
standard types, registered at runtime for third-party types). Comparison is
one 64-bit `==`. String aliases (`"yuv420p"`, `"I420"`, `"NV12"`) resolve to
the canonical TypeID via a name table. FourCC values are supported as aliases
for FFmpeg interop.

### Standard Property Keys

The framework defines standard keys for the built-in majors. Third-party
majors may define their own keys.

Property keys use **namespace:key** format for ecosystem clarity.

**Video** (`video:*`):

| Key | Type | Meaning |
|-----|------|---------|
| `video:width` | int | Frame width in pixels |
| `video:height` | int | Frame height in pixels |
| `video:pixel_format` | TypeID | Low-level pixel layout (redundant with `sub` for raw formats; useful when `sub` is a codec ID) |
| `video:color_space` | enum | BT.601, BT.709, BT.2020, … |
| `video:color_range` | enum | Limited (16–235) or Full (0–255) |
| `video:color_trc` | enum | Transfer characteristics (SDR, PQ, HLG, …) |
| `video:frame_rate` | rational | Frames per second as num/den |
| `video:sar` | rational | Sample (pixel) aspect ratio |
| `video:field_order` | enum | Progressive, TFF, BFF |

**Audio** (`audio:*`):

| Key | Type | Meaning |
|-----|------|---------|
| `audio:sample_rate` | int | Samples per second (e.g. 48000) |
| `audio:channels` | int | Number of channels |
| `audio:channel_layout` | bitmask | Speaker assignment (follows a defined enum) |
| `audio:sample_format` | TypeID | s16, s32, f32, f64, … |
| `audio:samples_per_frame` | int | Samples per buffer (may be variable) |

**Side-data** (`side:*`):

| Key | Type | Meaning |
|-----|------|---------|
| `side:codec` | TypeID | ASS, SRT, WebVTT, JSON, … |
| `side:language` | string | BCP 47 language tag |

**Namespace conventions:**

| Prefix | Owner | Example |
|--------|-------|---------|
| `video:`, `audio:`, `side:`, `tensor:` | Framework standard | `video:width` |
| `h264:`, `hevc:`, `av1:` | Codec-specific standard | `h264:profile` |
| `com.vendor:` | Third-party private | `com.acme:custom_score` |

### Extensibility

Third-party code registers new TypeIDs at runtime via
`registry.register("sens", "my_sensor_format")`. Property keys in custom
namespaces are free-form. Unknown namespaces are preserved and forwarded
through the pipeline.

### Capability Constraints

During format negotiation, pins advertise **capabilities** — sets of MediaType
templates with constraints rather than fixed values:

```
Capability {
    major: "vide",
    sub:   OneOf("NV12", "I420", "P010"),
    properties: {
        width:  Range(1, 8192),
        height: Range(1, 8192),
        color_space: OneOf(BT709, BT2020),
    }
}
```

Constraint types:

| Constraint | Meaning |
|------------|---------|
| `Exact(v)` | Must equal `v`. |
| `OneOf(v…)` | Must be one of the listed values. |
| `Range(lo, hi)` | Integer/rational must be in `[lo, hi]`. |
| `Any` | No constraint (accepts anything). |

Intersection of two capabilities produces a tighter capability or an empty set
(incompatible).

---

## Timestamp

A timestamp pairs a tick count with a time base.

```
Timestamp
├── ticks          : int64_t           // count in time-base units
└── time_base      : Rational          // time_base = num/den seconds per tick
```

**Sentinel:** `ticks == INT64_MIN` means "unknown / not set".

### Design Notes

The time-base model is the same one used by FFmpeg and Matroska. It avoids
precision loss when frame durations don't divide evenly into a fixed unit
(e.g. 1001/30000 s for 29.97 fps NTSC).

Each link negotiates its own time base as part of the `MediaType`. Frames on
the same link share a base; cross-link comparisons use `rescale()`. This
per-link model avoids precision loss for frame-rate-exact timestamps
(e.g. 1001/24000 s for 23.976 fps NTSC).

### Arithmetic

Utility functions for timestamp comparison, addition, and rescaling:

```cpp
Timestamp rescale(Timestamp ts, Rational new_base);
int       compare(Timestamp a, Timestamp b);  // handles different bases
Timestamp add(Timestamp ts, Timestamp delta);
double    to_seconds(Timestamp ts);
```

Rescaling uses 128-bit intermediate multiplication to avoid overflow.

