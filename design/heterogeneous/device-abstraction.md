# Device Abstraction

## Device

A `Device` identifies a specific compute unit in the system.

```
Device
├── provider_id       : TypeID            // registered provider type (same scheme as MediaType)
├── uuid              : DeviceUUID        // persistent hardware identifier (PCIe UUID, LUID, …)
├── index             : uint32_t          // session-local convenience index
├── physical_device_id: DeviceUUID        // shared across APIs for the same hardware
└── context           : opaque_ptr        // provider-specific runtime handle
```

- `provider_id` is a `TypeID`, same registration scheme as MediaType. String
  names like `"cuda"`, `"vulkan"` are convenience aliases.
- `uuid` is the canonical identifier — persistent across reboots and
  independent of enumeration order (unlike `index`, which can change with
  `CUDA_DEVICE_ORDER` or similar settings). CPU devices use a stable
  per-NUMA-node ID.
- `index` is a convenience for simple setups (`cuda:0`, `cuda:1`). Not used
  for equality or persistence.
- `physical_device_id` links Devices across APIs that share the same hardware.
  For example, a CUDA device and a Vulkan device on the same GPU share the
  same `physical_device_id`. The framework uses this for transfer-graph
  optimization (zero-copy interop edges) and resource contention awareness.

The CPU is modeled as a device (`provider_id = "cpu"`). The framework and
scheduler are agnostic to whether a device is a CPU or an accelerator — the
only structural property of the CPU is that host memory serves as the
universal transfer fallback, which is a transfer-graph property, not a
scheduling assumption.

### Device Equality

Two `Device` values are equal if and only if their `provider_id` and `uuid`
match. The `context` pointer is runtime state and not part of identity.

Devices from different providers with the same `physical_device_id` are not
equal but are known to share hardware. The transfer graph uses this to
register zero-copy interop edges between them.

---

## DeviceProvider

`DeviceProvider` is the extension point for adding new device backends. It is a
plugin-level interface — providers can be built into the framework (CPU, CUDA)
or loaded as plugins (Vulkan, OpenCL, Metal, …).

```cpp
class DeviceProvider {
public:
    virtual TypeID id() const = 0;

    virtual std::vector<Device> enumerate() = 0;

    virtual std::unique_ptr<BufferPool> create_pool(
        Device device, MediaType format, PoolConfig config) = 0;

    // Async copy between two buffers on this provider's devices.
    virtual Task<void> transfer_copy(
        const Buffer& src, Buffer& dst, TransferHint hint) = 0;

    // Create a read-only view of `src` accessible from `viewer_device`.
    // Returns nullopt if not supported for this device pair.
    virtual std::optional<ViewHandle> create_view(
        const Buffer& src, Device viewer_device) = 0;

    virtual void release_view(ViewHandle handle) = 0;

    virtual TransferCaps query_transfer(
        Device src_device, Device dst_device) = 0;

    virtual Task<void> sync(Device device, StreamHandle stream) = 0;

    // --- Virtual Memory (optional) ---

    virtual bool supports_virtual_memory() const { return false; }

    // Reserve a virtual address range without committing physical memory.
    virtual VMRange reserve(Device device, size_t size, size_t alignment) = 0;

    // Commit physical pages to a reserved range.
    virtual Status commit(VMRange range) = 0;

    // Release physical pages, keep the virtual range reserved.
    // Contents become undefined; pointer/handle remains stable.
    virtual void decommit(VMRange range) = 0;

    // Release the virtual range entirely.
    virtual void release(VMRange range) = 0;
};
```

`query_transfer` reports capabilities (can_copy, can_view, cost estimates) for
a given device pair. The transfer graph uses this at `prepare()` time.

### Virtual Memory Support

Providers that support virtual memory (`supports_virtual_memory() == true`)
enable the pool system to use reserve/commit/decommit instead of traditional
allocate/free. This provides stable addresses across reclamation cycles and
cheaper reuse after pressure (see
[Buffer Allocation — Virtual Memory Optimization](allocation.md#virtual-memory-optimization)).

| Provider | VM API | Notes |
|----------|--------|-------|
| cpu | `VirtualAlloc`/`mmap` + `madvise` | Universally available. |
| cuda | CUDA VMM (`cuMemAddressReserve`, `cuMemMap`) | Since CUDA 10.2, hardware-dependent. |
| vulkan | Sparse resources (`VK_BUFFER_CREATE_SPARSE_BINDING_BIT`) | Optional feature, not all devices. |

Providers without VM support use traditional allocate/free. The pool system
detects the capability per-provider and uses the appropriate path. Both paths
present the same interface to the rest of the framework.

### Provider SDK and Packaging

A device provider exports a **typed device wrapper** header that filter plugins
consume:

```cpp
// cuda_device.h — distributed as part of the CUDA provider SDK
class CUDADevice {
public:
    CUcontext get_context() const;
    CUstream  get_stream() const;
    // …
};

// In a filter:
auto& cuda = ctx.device().as<CUDADevice>();
CUcontext cu_ctx = cuda.get_context();
```

The provider SDK (headers + link stubs) is a separate distributable from the
provider's runtime plugin DLL. Filter plugins that target a specific device
API depend on that provider's SDK at build time.

### Provider Registration

Providers register themselves during plugin initialization:

```c
int32_t sdream_plugin_register(SDreamPluginRegistrar* r) {
    r->register_device_provider(&cuda_provider);
    return 0;
}
```

The framework calls `enumerate()` at startup and when devices are hot-plugged
(if supported by the OS).

---

## MemoryKind

Each buffer lives in a specific kind of memory. Memory kinds are
**provider-scoped** — CUDA pinned memory and Vulkan host-visible memory are
both "page-locked CPU memory," but they are not interchangeable (CUDA can't
DMA from Vulkan's host-visible allocation without explicit registration).

```
MemoryKind
├── provider_id  : TypeID          // which provider defines this kind
├── kind_id      : uint32_t        // provider-specific identifier
└── flags        : AccessFlags     // describes accessibility
```

The framework uses **flags**, not kind names, for transfer decisions.

### Access Flags

| Flag | Meaning |
|------|---------|
| `cpu_read` | CPU can read this memory. |
| `cpu_write` | CPU can write this memory. |
| `device_read` | The owning device can read. |
| `device_write` | The owning device can write. |
| `dma_capable` | Suitable as DMA source/destination (page-locked). |
| `coherent` | CPU and device see a consistent view without explicit flushes. |
| `migratable` | Driver may transparently migrate pages between CPU and device. |

### Standard Kinds per Provider

| Provider | Kind | Flags | Maps to |
|----------|------|-------|---------|
| cpu | `cpu:host` | cpu_rw | Standard `malloc`/aligned allocation. |
| cuda | `cuda:device` | device_rw | `cuMemAlloc` |
| cuda | `cuda:pinned` | cpu_rw, dma | `cuMemAllocHost` |
| cuda | `cuda:managed` | cpu_rw, device_rw, coherent, migratable | `cuMemAllocManaged` |
| vulkan | `vulkan:device` | device_rw | `DEVICE_LOCAL_BIT` |
| vulkan | `vulkan:host_visible` | cpu_rw, device_rw, dma, coherent | `HOST_VISIBLE_BIT \| HOST_COHERENT_BIT` |

Each provider registers its kinds during initialization. New providers
define their own kinds with appropriate flags.

### Framework Usage of Flags

- **Transfer needed?** Source flags lack the consumer device's access → transfer required.
- **DMA path?** Source has `dma_capable` → use DMA engine for copy.
- **Prefetch instead of copy?** Source has `migratable` → issue prefetch hint, no explicit copy.
- **View possible?** Provider reports `can_view` for this device pair.

`mmap`-ed memory (file-backed, shared memory) is just `cpu:host` — the
framework doesn't need to know the allocation strategy, only the accessibility
flags.

### CPU Provider

The CPU provider supports `cpu:host` (default, aligned allocation). Its
transfer is `memcpy`. It supports virtual memory on all platforms
(`VirtualAlloc`/`mmap`), enabling stable addresses and decommit-based
reclamation for CPU pools.

---

## Device Selection

When a filter declares that it can run on multiple device types (e.g.
`[CUDA, CPU]`), the framework must assign a concrete device. The algorithm:

1. **Prefer the user's explicit request** — the graph API allows pinning a
   filter to a specific device.
2. **Prefer minimizing transfers** — if neighbors are already assigned,
   prefer a device that matches the majority of connected filters.
3. **Prefer the filter's preference order** — the factory lists supported
   devices in decreasing preference.
4. **Fall back to the first available device** of the first supported type.

This is a heuristic; optimal device assignment in a general graph is
NP-hard (it reduces to graph partitioning). The heuristic runs once at
`prepare()` time and the assignment is fixed for the graph's lifetime.

---

## Device Context Sharing

Multiple filters on the same GPU share a device context (CUDA `CUcontext`,
Vulkan `VkDevice`) rather than each creating its own. The `DeviceProvider`
manages context lifetime:

- One context per `Device` instance.
- The `Device.context` opaque pointer provides access.
- Filters request the typed context via the provider SDK:

  ```cpp
  auto& cuda = ctx.device().as<CUDADevice>();
  CUcontext cu_ctx = cuda.get_context();
  ```

For Vulkan, the provider SDK additionally exposes `VkQueue` allocation among
filters that need exclusive queue access (e.g. compute vs transfer queues).

---

## Full and Hybrid Heterogeneous Filters

**Full heterogeneous:** The filter assumes all data is on its declared device.
The framework handles data migration on the links. This is the common case.

**Hybrid:** The filter manages both host and device work internally, using
`co_await ctx.on_device(...)` to migrate between thread groups. It may declare
different devices on input vs output pins (e.g. accept CPU packets, produce
GPU frames). The framework inserts transfers based on pin-level device
declarations — it does not need to know the filter is hybrid internally.

Both work under the existing design without special-casing.
