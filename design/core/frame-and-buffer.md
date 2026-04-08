# Frame & Buffer

## Frame

A frame is the unit of data flowing through the graph. It is a lightweight
handle — mostly metadata plus a reference-counted pointer to a buffer.

```
Frame
├── buffer         : intrusive_ptr<Buffer>  // payload (may be null for event-only frames)
├── pts            : Timestamp              // presentation time
├── dts            : Timestamp              // decode time (absent for raw formats)
├── duration       : Timestamp              // display/playback duration
├── media_type     : MediaType              // concrete format of this frame
├── flags          : FrameFlags             // bitfield — see below
└── props          : PropertyBag            // extensible metadata — see metadata.md
```

`intrusive_ptr` rather than `shared_ptr`: Buffer already carries `ref_count`,
so there is no need for a separate control block. One pointer, better cache
locality, no extra allocation.

### Frame Flags

| Flag | Meaning |
|------|---------|
| `Keyframe` | This frame is independently decodable. |
| `Corrupt` | Upstream detected corruption but produced a best-effort output. |
| `Discard` | Frame should not be displayed (e.g. preroll). |

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
├── memory_kind    : MemoryKind             // extensible per-provider tag
├── handle         : uint64_t               // opaque data handle (provider-specific)
├── size           : size_t                 // total allocation size in bytes
├── layout         : BufferLayout           // plane/stride description
├── pool           : PoolRef                // pool this buffer was allocated from (for recycling)
└── ref_count      : atomic<uint32_t>       // intrusive ref count
```

`handle` is `uint64_t` rather than `void*` — it signals "opaque, not
dereferenceable." The framework never interprets this; only filters and the
DeviceProvider do (cast to `CUdeviceptr`, `VkBuffer`, `void*`, etc.).

When the last reference is released, the buffer is returned to its pool rather
than freed — pool recycling is critical for avoiding repeated GPU allocations.

### BufferLayout

```
BufferLayout
├── planes         : SmallVector<PlaneDesc, 4>   // stack-alloc ≤4; heap for rare cases
├── plane[i]
│   ├── offset     : size_t                      // byte offset from base
│   ├── stride     : size_t                      // row pitch (bytes per row)
│   ├── rows       : uint32_t                    // number of rows in this plane
│   └── elem_size  : uint32_t                    // bytes per element
└── total_size     : size_t                      // ≥ max(plane offset + plane size)
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

### Single-Buffer Model

A frame references a single buffer. All practical GPU APIs (CUDA, Vulkan,
VAAPI, D3D11) represent multi-plane formats (NV12, P010) as a single
allocation with plane offsets. The single-buffer model is sufficient.

