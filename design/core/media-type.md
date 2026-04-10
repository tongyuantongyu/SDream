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

## Comments

> FourCC — don't like it as canonical representation. Evaluate: random UUIDs,
> custom structured IDs, enumerations.

**Response:**

| Option | Pros | Cons |
|--------|------|------|
| **Random UUIDs** | Globally unique, zero collision risk | Unreadable in debugger/logs (16 bytes of noise). Slow comparison (memcmp). |
| **Custom structured IDs** | Compact, partially readable, encodes category | Needs a registry to avoid collisions. Custom format to learn. |
| **Enumerations** | Fast, unambiguous, zero-cost comparison | Rigid — third-party types can't extend without recompiling. Terrible for extensibility. |

**Recommendation: structured 64-bit IDs.** Bits [63:48] = category (framework-defined: video, audio, side, tensor, …). Bits [47:0] = subtype (assigned by the framework for standard types, registered at runtime for third-party types).

- Comparison: one 64-bit `==`.
- Readable: `category` part tells you it's video/audio/etc. Subtype is a lookup.
- Extensible: third-party code calls `registry.register("my_format")` and gets a stable ID.
- String aliases: `"yuv420p"`, `"I420"`, `"NV12"` all resolve to the same ID via a name table. The canonical form is the 64-bit integer; strings are for human convenience and config files.

FourCC can be a supported alias format (auto-convert to canonical ID on registration) for FFmpeg interop.

---

> Property key namespacing — format-based vs namespace-based.

**Response:** Namespace-based (`domain:key`) is clearly better for ecosystem health:

- **`video:width`** — framework-standard video property. Every filter knows this.
- **`audio:sample_rate`** — framework-standard audio property.
- **`h264:qp_table`** — codec-specific standard. Registered by the H.264 plugin.
- **`com.vendor:custom_data`** — third-party private key. No collisions.

Impact on ecosystem:

- **Discoverability:** list all `video:*` keys to see the video property vocabulary.
- **Conflict-free:** `h264:profile` and `hevc:profile` coexist without ambiguity.
- **Forwarding rules:** framework can auto-forward all `video:*` keys through video transforms, drop codec-specific keys on transcode boundaries, and always forward `private:*` keys.
- **Cost:** string comparison is slightly longer, but property access is not in the frame-processing hot path. The namespace prefix can be stripped at registration time and replaced with a namespace ID for fast lookup internally.

Format-based naming (`_Standard` vs `Custom`) provides no structure and relies on convention that will inevitably be violated.

---

> Capability negotiation — flexible custom logic, standard implementation for
> common cases, auto-conversion only for lossless, width/height not negotiated.

**Response:** This is a significant refinement. Proposed design:

**1. Custom negotiation logic per filter.** The filter's `negotiate()` coroutine (already in the API) can implement arbitrary logic. The framework calls it during `prepare()`:

```cpp
auto fmts = co_await ctx.negotiate();
// fmts contains proposals from neighbors; filter can accept, counter-propose, or reject
```

**2. Standard implementation for common cases.** A `negotiate_standard(caps)` helper intersects declarative capability sets. Most filters call this and return. Filters with special needs override.

**3. Lossless-only auto-conversion.** The framework maintains a **safe conversion registry** — conversions that are known to be lossless:

| Conversion | Safe? | Reason |
|---|---|---|
| I420 ↔ NV12 | Yes | Same data, different plane layout |
| Host ↔ device transfer | Yes | Bit-exact copy |
| s16 → f32 (audio) | Yes | Widening, no precision loss |
| I444 → I420 | **No** | Chroma subsampling loses information |
| I420 → RGB | **No** | Color space conversion is lossy |
| Resolution change | **No** | Requires resampling |

Auto-converters are only inserted for safe conversions. Unsafe conversions require the user to explicitly add a converter filter. The safe registry is configurable: the user can promote "I420 → RGB" to safe for their pipeline if they accept the tradeoff.

**4. Width/height not auto-negotiated.** Agreed — spatial resolution is never auto-converted. If upstream and downstream disagree on resolution, negotiation fails with: *"Cannot negotiate: 'decode:out' produces 3840×2160 but 'encode:in' requires 1920×1080. Insert a scaler."* The graph builder (user or tool) is responsible for adding scalers explicitly.

This gives maximum flexibility to advanced filters while keeping simple filters simple (just call `negotiate_standard`).

