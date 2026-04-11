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

Each edge has a **cost** (latency + bandwidth estimate) and **capabilities**
(can_copy, can_view) reported by `query_transfer()`. To transfer from device A
to device B, the framework finds the cheapest path in this graph:

| Path type | Example | When chosen |
|-----------|---------|-------------|
| **View** | CUDA ↔ Vulkan on same GPU via `VK_KHR_external_memory` | Cheapest. Zero-copy. Provider or bridge reports `can_view`. |
| **Direct copy** | CUDA peer-to-peer between two GPUs | Low cost; provider reports direct capability. |
| **Staged via host** | CUDA → host `memcpy` → Vulkan upload | Always available as fallback; every provider handles `host`. |

If the cheapest path has multiple hops, multiple transfer filters are inserted
in sequence.

### Cost Overrides

Device plugins provide heuristic cost estimates, but the user may understand
their system topology better. The graph API exposes a cost override table:

```cpp
g.set_transfer_cost(cuda_0, vulkan_0, { .latency = 0, .bandwidth = INF });  // same GPU, fast
g.set_transfer_cost(cuda_0, cuda_1,   { .latency = 100, .bandwidth = 12 }); // PCIe, slow
```

User overrides take precedence over provider-reported costs. This lets users
tune transfer path selection for their specific hardware topology.

### Bridge Plugins

A bridge is a small plugin that registers transfer and view capabilities
between two device types without modifying either provider:

```c
int32_t sdream_plugin_register(SDreamPluginRegistrar* r) {
    static SDreamTransferBridge bridge = {
        .src_provider = "cuda",
        .dst_provider = "vulkan",
        .transfer_fn  = cuda_vulkan_interop_transfer,
        .create_view  = cuda_vulkan_interop_view,
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

A transfer filter is a framework-internal coroutine that handles cross-device
data movement. It chooses between **view mode** (zero-copy) and **copy mode**
based on capabilities and scheduler hints.

```cpp
Task device_transfer(FilterContext& ctx) {
    auto [in_fmt, out_fmt] = co_await ctx.negotiate();
    auto& provider = ctx.device_provider();
    bool use_view = can_view(src_device, dst_device) && scheduler_hint != PreferCopy;

    while (true) {
        Frame in = co_await ctx.pull(0);

        if (use_view) {
            Buffer* home = in.buffer()->home ? in.buffer()->home.get() : in.buffer();
            auto view_handle = provider.create_view(*home, dst_device);
            Frame out = wrap_view(view_handle, home, in);
            co_await ctx.push(0, std::move(out));
        } else {
            Frame out = co_await ctx.alloc(out_fmt, dst_device);
            co_await provider.transfer_copy(in.buffer(), out.buffer(), hint);
            out.copy_timing(in);
            co_await ctx.push(0, std::move(out));
        }
    }
}
```

**View creation always resolves to the home buffer** — if the input is itself
a view, the transfer creates a new view of the original home, not a view of a
view. This maintains the invariant that all views point directly to a home
buffer.

### Async DMA Overlap

On devices with independent DMA engines (CUDA copy engines, Vulkan transfer
queues), the transfer filter in copy mode submits the copy and suspends. The
scheduler resumes it when the copy completes (detected via event/fence).
Meanwhile, compute filters on the same GPU can continue running.

```
Time ──────────────────────────────────────▶

GPU compute:  ████ filter A ████   ████ filter A ████
GPU DMA:             ░░░ transfer ░░░
CPU scheduler: ─ dispatch A ─ dispatch transfer ─ dispatch A ─
```

Compute and DMA submissions use **one device thread** with separate
streams/queues. For CUDA, both a compute stream and a copy stream are
submitted from the same CPU thread. For Vulkan, the device thread submits to
separate queue families (compute vs transfer). The scheduler decides which
coroutine to resume on the device thread; the coroutine submits to its
appropriate stream/queue. Multiple device threads per device is a later
optimization if submission overhead becomes a bottleneck.

---

## Memory Pressure and Backpressure

### Physical-Device Memory Budget

Pools are budgeted at the **physical device** level, not per (provider,
device). A CUDA pool and a Vulkan pool on the same GPU compete for the same
VRAM. The framework maintains a memory budget per `physical_device_id` and all
pools sharing that physical device allocate from the same budget.

```
Physical GPU 0 (8 GB VRAM):
  cuda:0 pools:   using 3 GB
  vulkan:0 pools: using 2 GB
  total:          5 GB / 8 GB budget
```

### Per-Filter Buffer Tracking

The scheduler tracks how many buffers each filter is holding (directly or
via views). This enables targeted responses to memory pressure rather than
blunt global throttling.

### Backpressure as Primary Regulator

View-based retention is itself a backpressure mechanism. When views pin home
buffers, the home pool fills, and `alloc()` suspends the producer — naturally
throttling it to the consumer's rate. This is correct behavior, not a problem
to work around.

Switching views to copies under memory pressure is **not** a general-purpose
fix and is never done automatically:

- **Same physical device** (cross-API on same GPU): copy mode uses the same
  VRAM while wasting bandwidth on the copy. Actively harmful.
- **Different physical devices, consumer is sustainably slow:** copy mode just
  shifts buffer retention to the destination device. The destination pool fills
  eventually, and backpressure kicks in one stage later. Net effect: delayed
  throttling, not a throughput improvement.
- **GPU → CPU, temporary stall** (e.g. disk I/O hiccup): copy mode moves
  retention to CPU RAM (much larger), absorbing the stall. This is the one
  scenario where forcing copy mode can help — but it requires user judgment
  about the workload, so it is a manual graph-level override, not an
  automatic scheduler decision.

### Memory Pressure Response

When a device allocation fails with OOM, the pool system and scheduler
cooperate to free memory through an escalation chain — reclaiming free buffers
from sibling pools, shrinking FrameCache/fan-out buffering, and scaling down
filter clones. See
[Buffer Allocation and Memory Pressure](allocation.md) for the full
allocation flow, OOM response chain, and escalation details.

The framework does **not** automatically switch views to copies under memory
pressure. As analyzed above, automatic copy-switching is harmful or unhelpful
in most cases. Users who believe their specific workload benefits (e.g.
GPU → CPU with large CPU RAM headroom) can manually force copy mode on
individual transfer links via the graph construction API.

Backpressure (alloc suspension → producer throttling) remains the fundamental
regulator regardless of escalation.

### View/Copy Decision

The transfer filter chooses view or copy mode based on:

1. **Capability:** the provider/bridge reports `can_view` for this device pair.
2. **Consumer hint:** the downstream filter's input pin declares an
   `access_pattern` (see below).
3. **Physical device relationship:** same physical device → strongly prefer
   views (copy wastes bandwidth for no memory benefit).

Users can manually force copy mode on specific links via the graph
construction API (e.g. `g.force_copy(link)`) when they know their workload
benefits. The framework never switches views to copies automatically.

### Consumer Access Pattern Hints

A filter can declare an `access_pattern` on its input pins to influence the
transfer filter's view/copy decision:

| Pattern | Meaning | Transfer prefers |
|---------|---------|-----------------|
| `single_pass` (default) | Reads input once or twice. | View (zero-copy, one-time access cost is low). |
| `multi_pass` | Reads input many times (iterative shader, repeated sampling). | Copy (one DMA + fast local reads beats repeated remote reads). |

This is a declarative pin-level hint, not a runtime negotiation.

### Concrete View/Copy Decision Examples

| Scenario | Decision | Reason |
|----------|----------|--------|
| Same physical GPU, cross-API (CUDA→Vulkan) | **View** | Same VRAM, zero access cost. Always view — copy would waste bandwidth for no memory benefit. |
| Different GPUs with NVLink, consumer reads once | **View** | NVLink bandwidth is high. One read via peer access is cheaper than a full copy. |
| Different GPUs via PCIe, consumer reads many times | **Copy** | Repeated PCIe reads are slow. One DMA + fast local reads wins. |
| Consumer declares `multi_pass` | **Copy** | Hint overrides default — consumer needs fast repeated access. |
| Same device, different memory kind | **Copy** | Local DMA copy between memory kinds. |


---

## Managed Memory

Buffers with `memory_kind = managed` (CUDA unified memory, Vulkan shared
memory) are accessible from both CPU and device without explicit transfer.

When the transfer graph detects that both sides of a link can access managed
memory, the transfer filter may be replaced with a **prefetch filter** — a
lightweight coroutine that issues a prefetch hint to migrate the memory to the
consumer's device without copying:

```cpp
Task prefetch_transfer(FilterContext& ctx) {
    while (true) {
        Frame in = co_await ctx.pull(0);
        co_await provider.prefetch(in.buffer(), dst_device);
        co_await ctx.push(0, std::move(in));  // same frame, no copy
    }
}
```

The driver handles actual page migration. This avoids both explicit copies and
view creation overhead.

---

## Device-Aware Buffer Pools

Buffer pools are keyed by `(Device, MemoryKind, MediaType)` and share a
physical-device memory budget. The allocation flow, sibling pool reclamation,
OOM response chain, and pool sizing strategy are documented in
[Buffer Allocation and Memory Pressure](allocation.md).
