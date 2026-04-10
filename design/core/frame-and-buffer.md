# Frame & Buffer

## Frame

A frame is the unit of data flowing through the graph. It is a lightweight
handle — mostly metadata plus a reference-counted pointer to a buffer.

```
Frame
├── buffer         : shared_ptr<Buffer>     // payload (may be null for event-only frames)
├── pts            : Timestamp              // presentation time
├── dts            : Timestamp              // decode time (absent for raw formats)
├── duration       : Timestamp              // display/playback duration
├── media_type     : MediaType              // concrete format of this frame
├── flags          : FrameFlags             // bitfield — see below
└── props          : PropertyBag            // extensible metadata — see metadata.md
```

### Frame Flags

| Flag | Meaning |
|------|---------|
| `Keyframe` | This frame is independently decodable. |
| `Corrupt` | Upstream detected corruption but produced a best-effort output. |
| `Discard` | Frame should not be displayed (e.g. preroll). |
| `EOS` | Sentinel: no more frames on this stream. Carried as an event, not a normal frame. |

### Writable Access

Buffers are shared (reference-counted) for zero-copy fan-out. A frame pulled
from an input pin is therefore **read-only by default** — other links may hold
references to the same buffer.

Two helpers enable safe mutation:

```cpp
// Attempt in-place access. Returns the frame if sole owner, nullptr if shared.
Frame* inplace = ctx.try_make_writable(frame);

// Guarantee writable access. No-op if sole owner; copies if shared.
Frame writable = co_await ctx.make_writable(frame);
```

A filter that can process either in-place or out-of-place should prefer
`try_make_writable` — when it succeeds, the filter avoids both an allocation
and a copy. When it returns null, the filter falls back to allocating a
separate output buffer.

A filter that always needs to write calls `make_writable` directly. This is a
`co_await` because it may need to allocate from the buffer pool (which can
suspend if the pool is exhausted).

This design gives zero-copy fan-out and safe mutation without requiring a
framework-wide copy-on-write layer.

---

## Buffer

A buffer is a block of memory on a specific device, described by a layout.

```
Buffer
├── device         : DeviceRef              // owning device (CPU, CUDA:0, Vulkan:0, …)
├── memory_kind    : MemoryKind             // host / device / managed / pinned
├── base           : void*                  // base address (device pointer for GPU)
├── size           : size_t                 // total allocation size in bytes
├── layout         : BufferLayout           // plane/stride description
├── pool           : PoolRef                // pool this buffer was allocated from (for recycling)
└── ref_count      : atomic<uint32_t>       // shared ownership count
```

When the last reference is released, the buffer is returned to its pool rather
than freed — pool recycling is critical for avoiding repeated GPU allocations.

### BufferLayout

```
BufferLayout
├── planes         : uint32_t              // 1 for packed; 2–4 for planar formats
├── plane[i]
│   ├── offset     : size_t                // byte offset from base
│   ├── stride     : size_t                // row stride (bytes per row)
│   ├── height     : uint32_t              // rows in this plane
│   └── elem_size  : uint32_t              // bytes per element (pixel, sample)
└── total_size     : size_t                // ≥ max(plane offset + plane size)
```

This is general enough for:

- **Packed video** (RGB, YUYV): 1 plane.
- **Planar video** (I420, NV12): 2–3 planes.
- **Audio**: 1 plane for interleaved; N planes for planar (one per channel).
- **GPU textures**: `stride` maps to the device's row pitch (which may include
  alignment padding).

### Alignment

Buffer allocations respect device alignment requirements:

- CPU: configurable (default 64 bytes for SIMD).
- CUDA: `cuMemAlloc` handles alignment internally; pinned host memory aligned
  to page boundary.
- Vulkan: `VkMemoryRequirements::alignment`.

The pool records the alignment used so that recycled buffers satisfy the same
constraints.

### Multi-Buffer Frames

A frame references a single buffer. For formats that require separate
allocations (e.g. a GPU surface that stores luma and chroma in different
memory objects), use multi-plane layout within one allocation, or extend the
design with a `BufferList` if needed. The single-buffer model covers the vast
majority of formats.

## Comments

> pts/dts/duration as special fields vs per-frame metadata?

**Response:** Keep as special fields. The scheduler needs pts for QoS decisions on every frame. Sinks need pts for presentation timing. Muxers need pts/dts for interleaving. These are framework-level operations, not just filter-level — they need guaranteed-present, zero-cost access, not string-key lookup into a property bag. They're analogous to "length" on an array: technically metadata, but so fundamental that they belong on the struct.

Most filters do passthrough timestamps unchanged. The framework handles this automatically regardless of whether timestamps are fields or metadata. But fields make the framework's own use cheaper and type-safe.

---

> memory_kind — CUDA-centric? NUMA?

**Response:** MemoryKind is not a fixed enum — it's an extensible tag defined per-DeviceProvider. The `host/device/managed/pinned` in the doc are examples, not exhaustive. The CPU provider can define `host_numa0`, `host_numa1`, etc. The framework treats MemoryKind as opaque — it only uses it for pool keying and transfer path lookup.

For NUMA-aware scheduling: CPU workers can be bound to NUMA nodes, and the scheduler can prefer buffers from the matching memory kind. This is a scheduler-level optimization, not a core framework change. Worth noting in the design but not blocking.

---

> void* base — how to handle opaque handles (Vulkan VkBuffer, OpenGL textures)?

**Response:** Good catch. `void*` doesn't cover all APIs. The buffer data accessor should be opaque to the framework:

```
data: DeviceHandle    // provider-specific: void*, uint64_t, or opaque struct
```

The framework never dereferences this. Only filters and the DeviceProvider interpret it — a CUDA filter casts to `CUdeviceptr`, a Vulkan filter casts to `VkBuffer`. The framework's generic operations (ref counting, pool management, transfer dispatch) never touch the data handle.

This is a concrete design adjustment to make.

---

> shared_ptr vs intrusive_ptr?

**Response:** intrusive_ptr is strictly better here. Buffer already has `ref_count` as a member. intrusive_ptr gives: no separate control block allocation, one pointer instead of two, better cache locality. Clear win — recommend adopting.

---

> Plane merging efficiency?

**Response:** Plane merging (combining separate planes into one allocation) is rare. When needed, allocate a new buffer and copy. The copy cost is acceptable for a rare operation. Not worth framework-level support.

---

> Audio N planes — how large, fixed or SmallVector?

**Response:** Practical maximum: 8 (7.1 surround, planar). Immersive audio (Atmos, ambisonics) can go higher but typically uses interleaved format. Video: max 4 (YUV + alpha).

Recommendation: `SmallVector<4>` (stack-allocated for ≤4, heap for rare cases). Or a fixed array of 8 if you want to avoid any heap allocation in the layout struct.

---

> GPU surface with luma/chroma in different memory objects — real examples?

**Response:** In practice, separate allocations are uncommon. CUDA, Vulkan, VAAPI, and D3D11 all represent NV12/P010 surfaces as a single allocation with plane offsets. The "separate allocations" case is theoretical. The single-buffer model is sufficient — the `BufferList` mention can be removed.
