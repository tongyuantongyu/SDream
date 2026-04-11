# Heterogeneous Computing Model

SDream is designed from the ground up to support graphs that mix filters
running on different compute devices — CPU, CUDA GPUs, Vulkan GPUs, and
extensible to any future accelerator.

The key principle: **the user builds a logical graph without worrying about
data transfers. The framework detects cross-device links and inserts the
necessary migration automatically.**

Both "full" heterogeneous filters (all data on device, all processing on
device) and "hybrid" filters (mixing host and device work internally via
`on_device()`) work under the existing design without special-casing. See
[Device Abstraction — Full and Hybrid Filters](device-abstraction.md#full-and-hybrid-heterogeneous-filters).

## Sub-Pages

| Document | Contents |
|----------|----------|
| [Device Abstraction](device-abstraction.md) | Device model, DeviceProvider interface, MemoryKind, context sharing |
| [Data Migration](data-migration.md) | View-based transfers, transfer path resolution, bridge interop, view/copy decisions |
| [Buffer Allocation](allocation.md) | Buffer pool architecture, allocation flow, OOM response chain, pool sizing |
