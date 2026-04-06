# Data Migration

## Automatic Transfer Insertion

During `prepare()`, after device assignment and format negotiation, the
framework inspects every link:

```
for each link (src_pin → dst_pin):
    if src_pin.device != dst_pin.device:
        path = resolve_transfer_path(src_pin.device, dst_pin.device)
        insert transfer filter(s) on this link using `path`
```

The inserted transfer filters are part of the **physical graph** — invisible
in the logical graph the user constructed but visible in debugging/profiling
views.

---

## Transfer Path Resolution

The framework builds a **transfer graph** at startup from all registered
`DeviceProvider`s and bridge plugins:

```
  ┌──────┐  host↔device   ┌──────┐
  │ CPU  │◄──────────────▶│ CUDA │
  └──┬───┘                └──┬───┘
     │ host↔device           │ CUDA-Vulkan interop (bridge plugin)
  ┌──┴───┐                ┌──┴───┐
  │Vulkan│◄───────────────│(brg) │
  └──────┘                └──────┘
```

Each edge has a **cost** (latency + bandwidth estimate) reported by
`query_transfer()`. To transfer from device A to device B, the framework finds
the cheapest path in this graph:

| Path type | Example | When chosen |
|-----------|---------|-------------|
| **Direct** | CUDA peer-to-peer between two GPUs | Lowest cost; provider reports direct capability. |
| **Bridge** | CUDA → Vulkan via `VK_KHR_external_memory` interop | Bridge plugin registered a zero-copy or low-cost edge. |
| **Staged via host** | CUDA → host `memcpy` → Vulkan upload | Always available as fallback; every provider handles `host`. |

If the cheapest path has multiple hops, multiple transfer filters are inserted
in sequence.

### Bridge Plugins

A bridge is a small plugin that registers a transfer capability between two
device types without modifying either provider:

```c
int32_t sdream_plugin_register(SDreamPluginRegistrar* r) {
    static SDreamTransferBridge bridge = {
        .src_provider = FOURCC("CUDA"),
        .dst_provider = FOURCC("VLKN"),
        .transfer_fn  = cuda_vulkan_interop_transfer,
        .query_fn     = cuda_vulkan_interop_query,
    };
    r->register_transfer_bridge(&bridge);
    return 0;
}
```

This keeps provider implementations independent and allows the interop
surface to evolve separately.

---

## Transfer Filter

A transfer filter is a framework-internal coroutine:

```cpp
Task device_transfer(FilterContext& ctx) {
    auto [in_fmt, out_fmt] = co_await ctx.negotiate();
    auto& provider = ctx.device_provider();

    while (true) {
        Frame in = co_await ctx.pull(0);
        if (in.is_eos()) break;

        Frame out = co_await ctx.alloc(out_fmt, dst_device);
        co_await provider.transfer(in.buffer(), out.buffer(), hint);
        out.copy_timing(in);
        out.copy_props(in);

        co_await ctx.push(0, std::move(out));
    }
}
```

The `co_await provider.transfer(…)` call may wait for a DMA engine or a GPU
copy kernel. While it waits, the scheduler can run other filters — this is how
compute/transfer overlap happens.

### Async DMA Overlap

On devices with independent DMA engines (CUDA copy engines, Vulkan transfer
queues), the transfer filter submits the copy and suspends. The scheduler
resumes it when the copy completes (detected via event/fence). Meanwhile,
compute filters on the same GPU can continue running.

```
Time ──────────────────────────────────────▶

GPU compute:  ████ filter A ████   ████ filter A ████
GPU DMA:             ░░░ transfer ░░░
CPU scheduler: ─ dispatch A ─ dispatch transfer ─ dispatch A ─
```

---

## Device-Aware Buffer Pools

Buffer pools are keyed by `(Device, MediaType)`:

```
Pool key: (CUDA:0, {vide, NV12, 1920×1080})
  ├── allocated: 8 buffers
  ├── free:      3 buffers
  └── capacity:  8 (max)
```

Pool behavior:

1. `alloc()` → return a free buffer from the matching pool.
2. No free buffer and pool below capacity → allocate via the device provider.
3. Pool at capacity → suspend the caller until a buffer is returned.

Pools are created lazily (on first `alloc` with a new key) and destroyed when
the graph is torn down. Pool capacity is configurable per-pool or via a global default.

### Pool Sizing

Pool capacity is bounded to prevent unbounded GPU memory consumption. A
reasonable default is `queue_capacity + 1` for each link using that pool —
this guarantees that the pipeline can fill all queues without deadlock while
limiting peak memory to a predictable multiple of frame size.
