# Heterogeneous Computing Model

SDream is designed from the ground up to support graphs that mix filters
running on different compute devices — CPU, CUDA GPUs, Vulkan GPUs, and
extensible to any future accelerator.

The key principle: **the user builds a logical graph without worrying about
data transfers. The framework detects cross-device links and inserts the
necessary migration automatically.**

## Sub-Pages

| Document | Contents |
|----------|----------|
| [Device Abstraction](device-abstraction.md) | Device model, DeviceProvider interface, MemoryKind, context sharing |
| [Data Migration](data-migration.md) | Auto-migration, transfer path resolution, bridge interop, device-aware pools |
