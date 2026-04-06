# Device Abstraction

## Device

A `Device` identifies a specific compute unit in the system.

```
Device
├── provider_id    : FourCC          // "CPU_", "CUDA", "VLKN", "OPCL", …
├── index          : uint32_t        // distinguishes multiple devices of the same type
└── context        : opaque_ptr      // provider-specific runtime handle
```

The CPU is modeled as a device (`provider_id = "CPU_"`, `index = 0`). This
uniformity means the framework's transfer and scheduling logic does not
special-case the CPU.

### Device Equality

Two `Device` values are equal if and only if their `provider_id` and `index`
match. The `context` pointer is runtime state and not part of identity.

---

## DeviceProvider

`DeviceProvider` is the extension point for adding new device backends. It is a
plugin-level interface — providers can be built into the framework (CPU, CUDA)
or loaded as plugins (Vulkan, OpenCL, Metal, …).

```cpp
class DeviceProvider {
public:
    virtual FourCC id() const = 0;

    // Enumerate available devices of this type.
    virtual std::vector<Device> enumerate() = 0;

    // Create a buffer pool for the given device and format.
    virtual std::unique_ptr<BufferPool> create_pool(
        Device device, MediaType format, PoolConfig config) = 0;

    // Async transfer between two buffers.
    // `src` and `dst` may be on different memory kinds of the same device,
    // or `src` may be on host and `dst` on this provider's device (or vice versa).
    virtual Task<void> transfer(
        const Buffer& src, Buffer& dst, TransferHint hint) = 0;

    // Query whether this provider can perform a transfer from `src_device`
    // to `dst_device`, and at what cost.
    virtual TransferCaps query_transfer(
        Device src_device, Device dst_device) = 0;

    // Wait for all pending async work on a device stream/queue.
    virtual Task<void> sync(Device device, StreamHandle stream) = 0;
};
```

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

Each buffer lives in a specific kind of memory on a specific device.

| Kind | Meaning | Required? |
|------|---------|-----------|
| `host` | Ordinary CPU-accessible RAM. | Every provider must support host ↔ device transfer. |
| `device` | Accelerator-local memory, not CPU-accessible. | Typical for CUDA, Vulkan. |
| `managed` | Unified memory, coherently accessible by both CPU and device. | CUDA managed memory, shared Vulkan memory. |
| `host_pinned` | Page-locked host memory, faster for DMA. | CUDA, Vulkan host-visible. |

Providers may define additional kinds. The framework does not hard-code these —
it uses them only for transfer path resolution.

### CPU Provider

The CPU provider supports a single memory kind: `host`. Its "transfer" is a
`memcpy`. Its "pool" is a standard aligned-allocation free-list.

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

Multiple filters on the same GPU should share a device context (CUDA
`CUcontext`, Vulkan `VkDevice`) rather than each creating its own. The
`DeviceProvider` manages context lifetime:

- One context per `Device` instance.
- The `Device.context` opaque pointer provides access.
- Filters request the typed context from the provider at setup time:

  ```cpp
  auto cu_ctx = ctx.device().as<CUcontext>();
  ```

For Vulkan, the provider additionally manages `VkQueue` allocation among
filters that need exclusive queue access (e.g. compute vs transfer queues).
