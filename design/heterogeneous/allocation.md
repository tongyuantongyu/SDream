# Buffer Allocation and Memory Pressure

This document describes the unified buffer pool system, workspace allocation,
virtual memory optimization, and the OOM response chain.

---

## Pool Architecture

All memory allocations — pipeline buffers, workspaces, and any other
framework-managed memory — go through a unified pool system. Pools are keyed
by `(Device, MemoryKind, AllocationSpec)`:

```
AllocationSpec
├── size       : size_t       // total allocation size
├── alignment  : size_t       // device alignment requirement
└── layout     : BufferLayout // optional — present for pipeline buffers, absent for workspaces
```

Pipeline buffer allocations derive their `AllocationSpec` from the negotiated
`MediaType` and the device's alignment rules. Workspace allocations specify
size and alignment directly (no layout).

```
Pool examples:
  (cuda:0, cuda:device, {4MB, 512, NV12 1920×1080})   — pipeline buffer pool
  (cuda:0, cuda:device, {16MB, 512, —})                — workspace pool (16MB size class)
  (cpu:0,  cpu:host,    {1MB, 64, —})                  — workspace pool (1MB size class)
```

All pools on the same `physical_device_id` share a **physical-device memory
budget** — a soft ceiling on total committed bytes.

Pools are created lazily on first allocation with a new key and destroyed when
the graph is torn down.

---

## Virtual Memory Optimization

When the `DeviceProvider` supports virtual memory (see
[DeviceProvider — Virtual Memory](../heterogeneous/device-abstraction.md#virtual-memory-support)),
pools use a reserve/commit/decommit model instead of traditional
allocate/free:

| Operation | Traditional | VM-backed |
|-----------|------------|-----------|
| **Pool entry created** | Allocate from provider | Reserve virtual range, commit physical pages |
| **Buffer returned to pool** | Keep allocated (fast reuse) | Keep committed (fast reuse) — identical fast path |
| **Reclamation under pressure** | Deallocate (full free to provider) | Decommit pages (release physical memory, keep virtual range) |
| **Reuse after reclamation** | Re-allocate from provider (expensive) | Recommit pages in existing range (cheaper, no provider call) |

**Key properties:**

- **Fast path is identical.** Grabbing a committed buffer from the free list
  and returning it are pointer operations in both models.
- **Stable addresses.** The virtual range persists across decommit/recommit
  cycles. Filters see the same pointer (or device handle), though contents are
  undefined after recommit.
- **Cheaper recovery.** Recommitting an existing virtual range avoids a full
  provider allocation call. For GPU devices with expensive allocation APIs,
  this difference is significant.

For providers without VM support, pools fall back to traditional
allocate/free. The rest of the framework (filters, scheduler, OOM response)
sees the same interface regardless.

**CPU provider:** uses `VirtualAlloc(MEM_RESERVE/MEM_COMMIT)` on Windows,
`mmap` + `madvise(MADV_DONTNEED)` on Linux.

**GPU providers:** CUDA VMM (`cuMemAddressReserve` + `cuMemMap`) since CUDA
10.2, Vulkan sparse resources. Capability varies by hardware — the provider
reports support via `supports_virtual_memory()`.

---

## Allocation Flow

When a filter calls `ctx.alloc(...)` or `ctx.alloc_workspace(...)`:

```
alloc(spec)
  │
  ├─ 1. Committed free buffer in matching pool?  ──yes──→  return it  (fast path)
  │
  ├─ 2. Decommitted entry in matching pool? (VM-backed only)
  │     ──yes──→  recommit pages
  │               ├─ success → return buffer
  │               └─ OOM     → enter OOM response
  │
  ├─ 3. Physical-device budget allows new allocation?
  │     │
  │     ├─ yes → allocate from provider (or reserve + commit if VM)
  │     │         ├─ success → return buffer
  │     │         └─ OOM     → enter OOM response
  │     │
  │     └─ no  → try reclamation (step 4)
  │
  ├─ 4. Reclaim from sibling pools on same physical device.
  │     ├─ VM: decommit free (committed) buffers in sibling pools
  │     └─ Traditional: deallocate free buffers in sibling pools
  │     │
  │     ├─ freed enough → retry from step 3
  │     └─ nothing reclaimable → enter OOM response
  │
  └─ 5. Suspend caller (backpressure).
        Coroutine suspends until a buffer is returned to any pool on this
        physical device, or until an OOM escalation step frees memory.
```

Steps 1–4 happen synchronously. Step 5 engages the coroutine scheduler.

### Sibling Pool Reclamation

"Sibling pools" are all pools sharing the same `physical_device_id`. A pool's
free committed buffers are reclamation candidates.

Reclamation priority (heuristic):

- Pools with the **lowest recent hit rate** are reclaimed first.
- Pools whose spec differs from the requesting pool are preferred (reclaiming
  a pool's own free buffers defeats the purpose of pooling).
- Never reclaim in-use buffers — those are held by filters and will return
  naturally via backpressure.

---

## Workspace Allocation

Filters often need private scratch memory — intermediate arrays, lookup
tables, temporary computation buffers. The framework manages this as
**workspaces**, giving the scheduler visibility into memory that would
otherwise be invisible.

### API

```cpp
auto ws = co_await ctx.alloc_workspace(size);                     // unlocked, on current device
auto ws = co_await ctx.alloc_workspace(size, WorkspaceMode::Locked);  // locked
auto ws = co_await ctx.alloc_workspace(size, device);             // on specific device

void* ptr = ws.data();     // current pointer (stable if VM-backed)

ws.lock();                 // promote to locked
ws.unlock();               // demote to unlocked
ws.release();              // explicit release (also released on handle destruction)
```

Workspace allocations go through the same pool system as pipeline buffers.
The pool key uses the workspace's `(Device, MemoryKind, size/alignment)` —
workspaces of similar sizes on the same device reuse the same pool.

### Modes

**Unlocked (default).** The framework does not preserve workspace contents
across `co_await` points. While the filter is suspended, the scheduler *may*
reclaim the workspace's physical memory. The filter must treat contents as
undefined after any `co_await`.

- Under normal conditions (no pressure), the pool keeps the memory committed
  for fast reuse — the "no preservation" contract gives the framework the
  *right* to reclaim, not an obligation.
- Under memory pressure, unlocked workspaces of suspended filters are a
  reclamation source (see OOM Response below).
- If reclaimed and the filter resumes, the workspace is recommitted (VM) or
  re-allocated (traditional). Contents are undefined.
- If the provider supports VM, the workspace pointer remains stable across
  reclaim/recommit. Otherwise it may change — filters must re-fetch via
  `ws.data()` after any `co_await` for unlocked workspaces.

**Locked.** Contents are preserved across `co_await` points. The framework
cannot reclaim the memory, but knows it is idle while the filter is suspended.
The scheduler factors locked workspace sizes into scheduling decisions (e.g.,
avoid running too many filters with large locked workspaces concurrently).

Future optimization: under extreme GPU memory pressure, the scheduler could
**spill** locked workspaces to host memory (DMA out on suspend, DMA back on
resume). This is not implemented initially — backpressure and scheduling
awareness handle the common cases.

### Interaction with Filter Cloning

Each clone of an auto-parallelized filter gets its own workspace. The memory
cost of cloning scales as `N × workspace_size`. The scheduler factors this
into clone count decisions — a filter with a large workspace has a higher
per-clone memory cost, reducing the practical maximum N.

---

## OOM Response

When the OS/driver returns an OOM error, the pool system and scheduler
cooperate to free memory. This is triggered by a real allocation failure, not
a speculative budget check.

### Escalation Steps

1. **Reclaim sibling pool free buffers.** (Already attempted in the allocation
   flow. Skipped if the OOM came after reclamation.)

2. **Reclaim unlocked workspaces.** Decommit (VM) or deallocate (traditional)
   unlocked workspaces held by *suspended* filters on the pressured device.
   These are idle memory that the framework has explicit permission to reclaim.

3. **Reduce buffering.** The scheduler shrinks fan-out buffer sizes and
   FrameCache capacities for filters on the pressured device. As these filters
   process their current frames, the excess buffers flow back to pools.

4. **Scale down filter clones.** Reduce the clone count of auto-parallelized
   filters (see
   [Scheduler — Filter Cloning](../execution/scheduler.md#filter-cloning-automatic-parallelization))
   on the pressured device. Fewer clones mean fewer in-flight buffers and
   fewer workspaces.

5. **Serialize execution.** Reduce scheduler thread parallelism to lower peak
   buffer concurrency across the pipeline.

The framework does **not** automatically switch views to copies under memory
pressure — in most cases this is harmful or merely shifts the problem. Users
can manually force copy mode on specific links via the graph construction API
(see
[Data Migration — Backpressure as Primary Regulator](data-migration.md#backpressure-as-primary-regulator)).

### Escalation Timing

Steps 2–5 do not free memory instantaneously. Step 2 takes effect immediately
(the workspace memory is under framework control). Steps 3–5 reduce the
*steady-state* buffer count, taking effect over subsequent frames.

The pool system retries after each step:

1. Apply the escalation.
2. Suspend the blocked caller.
3. Resume when a buffer is returned to any pool on the pressured device.
4. Retry allocation. If still OOM, escalate to the next step.

If all steps are exhausted, the caller remains suspended. Backpressure
propagates upstream until the pipeline drains enough buffers.

### Interaction with Backpressure

Backpressure (alloc suspension → producer throttling) is the fundamental
regulator regardless of escalation. The escalation steps reduce *peak* buffer
demand so that OOM-triggered suspensions become less frequent in steady state.

---

## Pool Sizing

The scheduler sizes pools based on:

- **Pipeline depth:** the number of buffers in flight within the pipeline
  segment using this pool.
- **View retention:** home buffers held alive by downstream views are not
  reclaimable until views release.
- **Workspace retention:** locked workspaces are non-reclaimable while the
  filter exists. Unlocked workspaces are soft — reclaimable under pressure.
- **Physical budget:** all pools on the same `physical_device_id` share
  a memory ceiling.

Pool sizing is advisory — pools can grow beyond their soft limit if the
physical budget allows, and shrink under OOM pressure. The soft limits guide
the sibling reclamation heuristic: pools well above their target size are
reclaimed first.
