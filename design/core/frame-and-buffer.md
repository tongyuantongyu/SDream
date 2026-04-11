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

Buffers are shared (reference-counted) for zero-copy fan-out and cross-device
views. A frame pulled from an input pin is therefore **read-only by default**.

Two helpers enable safe mutation:

```cpp
// Attempt in-place access. Returns the frame if sole owner, nullptr if shared.
Frame* inplace = ctx.try_make_writable(frame);

// Guarantee writable access. No-op if sole owner; copies if shared.
Frame writable = co_await ctx.make_writable(frame);
```

**Behavior:**
- **Home buffer, ref_count == 1:** sole owner, no views — return as-is (writable
  in place).
- **Home buffer, ref_count > 1:** shared with other frames or views — copy to a
  fresh buffer on the same device.
- **View buffer:** always copy to a fresh home buffer on the viewer's device.
  The copy is a local operation — the view's handle is a valid handle on the
  viewer's device, so the device's copy command handles the data path
  internally.

A filter that can process either in-place or out-of-place should prefer
`try_make_writable` — when it succeeds, the filter avoids both an allocation
and a copy. When it returns null, the filter falls back to allocating a
separate output buffer. `try_make_writable` returns null for views
(since they always require a copy).

This design gives zero-copy fan-out, zero-copy cross-device reads, and safe
mutation without requiring a framework-wide copy-on-write layer.

---

## Buffer

A buffer is a block of memory on a specific device, described by a layout.

```
Buffer
├── device         : DeviceRef              // device this handle is valid on
├── memory_kind    : MemoryKind             // framework-defined memory category
├── handle         : uint64_t               // opaque data handle (provider-specific)
├── size           : size_t                 // total allocation size in bytes
├── layout         : BufferLayout           // plane/stride description
├── pool           : PoolRef                // pool this buffer was allocated from (null for views)
├── home           : intrusive_ptr<Buffer>  // null = home buffer; non-null = view of home
└── ref_count      : atomic<uint32_t>       // intrusive ref count
```

`handle` is `uint64_t` rather than `void*` — it signals "opaque, not
dereferenceable." The framework never interprets this; only filters and the
DeviceProvider do (cast to `CUdeviceptr`, `VkBuffer`, `void*`, etc.).

### Home Buffers and Views

A **home buffer** (`home == nullptr`) is a real allocation, managed by a
buffer pool. When the last reference is released, it is returned to its pool
rather than freed — pool recycling is critical for avoiding repeated GPU
allocations.

A **view buffer** (`home != nullptr`) is a mapping of a home buffer's memory
into a different device's address space. It holds a strong ref to the home,
keeping the underlying allocation alive. Views are created by the transfer
system for zero-copy cross-device access (see
[Data Migration](../heterogeneous/data-migration.md)).

**Invariants:**
- `home` always points to a home buffer (never to another view). No view chains.
- Views are **read-only**. Filters that need to write must call `make_writable()`,
  which copies to a fresh home buffer.
- A home buffer's ref_count includes refs from views. `ref_count == 1` on a
  home buffer implies no outstanding views.
- When a view is destroyed: the device mapping is released, then the home ref
  is dropped (which may return the home to its pool).
- `alloc()` always returns home buffers. Views are only created by the
  framework's transfer system.

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

### Future: Copy-on-Write

With VM-backed pools (see
[Buffer Allocation — Virtual Memory](../heterogeneous/allocation.md#virtual-memory-optimization)),
OS-level copy-on-write becomes feasible for CPU buffers: instead of an
explicit copy in `make_writable()`, the OS can remap pages with COW semantics,
deferring the copy to the first write. This is a transparent optimization
under the existing `make_writable()` API — no design changes needed.

---

## Workspace

A workspace is a private scratch allocation owned by a single filter. Unlike
frame buffers, workspaces do not flow through the pipeline — they are internal
to a filter's processing logic (intermediate arrays, lookup tables, temporary
computation buffers).

```
Workspace
├── device      : DeviceRef      // device this memory resides on
├── memory_kind : MemoryKind     // memory category
├── handle      : uint64_t       // opaque data handle (same convention as Buffer)
├── size        : size_t         // allocation size in bytes
├── mode        : WorkspaceMode  // Locked or Unlocked
└── filter      : FilterRef      // owning filter (tracked by scheduler)
```

Workspaces are allocated through the same unified pool system as frame
buffers (see
[Buffer Allocation](../heterogeneous/allocation.md)).

**Unlocked (default):** Contents are not preserved across `co_await` points.
The framework may reclaim the physical memory while the filter is suspended.
Filters must treat contents as undefined after resume. If the provider
supports virtual memory, the pointer (`ws.data()`) remains stable across
reclaim/recommit cycles; otherwise it may change.

**Locked:** Contents are preserved across `co_await` points. The framework
cannot reclaim the memory, but knows it is idle while the filter is suspended
and factors this into scheduling decisions.

See [Buffer Allocation — Workspace Allocation](../heterogeneous/allocation.md#workspace-allocation)
for the full allocation API, reclamation behavior, and memory pressure
interactions.

