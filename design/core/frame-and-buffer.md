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

A frame pulled from an input pin is **read-only by default** because other
links (fan-out) may hold references to the same buffer. A filter that needs to
modify the data calls:

```cpp
Frame writable = co_await ctx.make_writable(frame);
```

- If `ref_count == 1`, this is a no-op (returns the same frame).
- Otherwise, it allocates a new buffer, copies the data, and returns a frame
  referencing the private copy.

This pattern is the practical realization of **[D04]** and **[D14]** — it
gives both zero-copy fan-out and safe mutation without requiring a
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

## Questions

~~No active questions~~.

> ├── pts            : Timestamp              // presentation time
├── dts            : Timestamp              // decode time (absent for raw formats)
├── duration       : Timestamp              // display/playback duration

Would these timestamps better be special fields or just be one of the per-frame metadata?

At least in video and audio world, most filters would either don't care or simply assume the fps is steady.

---

> ├── memory_kind    : MemoryKind             // host / device / managed / pinned



---

> ├── base           : void*                  // base address (device pointer for GPU)
├── size           : size_t                 // total allocation size in bytes

How does this handle APIs that gives opaque handles?

---

> ├── buffer         : shared_ptr<Buffer>     // payload (may be null for event-only frames)

Would it be better to invent our intrusive_ptr?

---

> │   ├── offset     : size_t                // byte offset from base

Is it worth the effort to support plane-merging actions more efficiently (we are forced to reallocate with current design)?

---

> **Audio**: 1 plane for interleaved; N planes for planar (one per channel).

How crazy can N become? Can we use fixed N or consider `SmallVector`?

---

> e.g. a GPU surface that stores luma and chroma in different
memory objects

Are there real world examples?

---

