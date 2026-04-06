# Media Type & Timestamp

## MediaType

A MediaType is the concrete description of data format on a link. It combines
a structured type tag with an extensible property map.

```
MediaType
├── major          : FourCC             // "vide", "audi", "side", …
├── sub            : FourCC             // "NV12", "I420", "FL32", "ASS_", …
└── properties     : PropertyMap        // key → Value (int, float, rational, string, blob)
```

### Standard Property Keys

The framework defines standard keys for the built-in majors. Third-party
majors may define their own keys.

**Video** (`major = "vide"`):

| Key | Type | Meaning |
|-----|------|---------|
| `width` | int | Frame width in pixels |
| `height` | int | Frame height in pixels |
| `pixel_format` | fourcc | Low-level pixel layout (redundant with `sub` for raw formats; useful when `sub` is a codec ID) |
| `color_space` | enum | BT.601, BT.709, BT.2020, … |
| `color_range` | enum | Limited (16–235) or Full (0–255) |
| `color_trc` | enum | Transfer characteristics (SDR, PQ, HLG, …) |
| `frame_rate` | rational | Frames per second as num/den |
| `sar` | rational | Sample (pixel) aspect ratio |
| `field_order` | enum | Progressive, TFF, BFF |

**Audio** (`major = "audi"`):

| Key | Type | Meaning |
|-----|------|---------|
| `sample_rate` | int | Samples per second (e.g. 48000) |
| `channels` | int | Number of channels |
| `channel_layout` | bitmask | Speaker assignment (follows a defined enum) |
| `sample_format` | fourcc | s16, s32, f32, f64, … |
| `samples_per_frame` | int | Samples per buffer (may be variable) |

**Side-data** (`major = "side"`):

| Key | Type | Meaning |
|-----|------|---------|
| `codec` | fourcc | ASS, SRT, WebVTT, JSON, … |
| `language` | string | BCP 47 language tag |

### Extensibility

The `major` and `sub` fields are FourCC values — 4-byte identifiers.
Third-party code can introduce new majors (e.g. `"sens"` for sensor data)
and new subs without modifying the framework core. Property keys are strings;
new keys can be added freely. Unknown keys are preserved and forwarded.

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

## Questions

> ├── major          : FourCC             // "vide", "audi", "side", …
├── sub            : FourCC             // "NV12", "I420", "FL32", "ASS_", …

I don't quite like FourCC, at least not as the canonical representation.

I'm evaluating:

1. Assigning them random uuids.

2. Assigning them custom ids (may or may not in uuid size), with type-indication part and subtype value part (which can be either random assigned or represent characteristics)

3. Enumerations.

---

> Standard Property Keys

I'm considering differentiate property sources, here are the ideas:

1. `Standard`, `custom`
2. `_Standard`, `Custom`, `custom`
3. `standard`, `codec:custom`
4. `video:standard`, `custom`

Compare the plans, focus on the impact to filter ecosystem and interop.

---

