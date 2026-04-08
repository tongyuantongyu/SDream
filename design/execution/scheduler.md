# Scheduler

The scheduler is the framework's control-flow engine. It maintains a set of
filter coroutines, tracks their dependencies, and dispatches ready ones for
execution.

## Responsibilities

1. **Dependency tracking.** When a coroutine suspends (on pull, push, alloc,
   …), the scheduler records what it is waiting for.
2. **Readiness detection.** When a resource becomes available (frame enqueued,
   queue space freed, buffer returned to pool), the scheduler moves the
   affected coroutine(s) to the ready set.
3. **Dispatch.** Select a ready coroutine and resume it on a worker thread.
4. **Backpressure enforcement.** Respect queue capacity limits — never wake a
   producer if all its output queues are full.
5. **Termination.** Detect when the entire graph has finished (all filters
   Done or Error) and signal completion.

## Core Loop (Conceptual)

```
while graph is not done:
    wait until ready_set is non-empty
    coro = select(ready_set)          // scheduling policy
    worker = pick_thread(coro)        // threading + device affinity
    resume coro on worker
    // coro runs until it hits co_await …
    // co_await handler registers dependency and returns here
    update dependency graph
    move newly-ready coroutines into ready_set
```

In a single-threaded configuration, `pick_thread` always returns the current
thread. In a thread-pool configuration, it dispatches to a free worker.

## Scheduling Policy (Pluggable)

The `select(ready_set)` step is the scheduling strategy. Multiple scheduler
implementations coexist, each tuned for different use cases. The application
selects one when building the graph.

| Scheduler | Strategy | Best for |
|-----------|----------|----------|
| **Max utilization** | Wake filters aggressively, maximize pipeline parallelism. Multi-threaded, auto-inserts FrameCache (FIFO) generously. | Offline transcoding, batch processing — saturates hardware. |
| **Constant speed** | Pace sources to match sink's real-time rate. Monitors pipeline clock; throttles sources when output latency exceeds target. | Playback, live streaming — regulates latency and bufferbloat. |
| **Blueprint** | Execute filters in a fixed topological order, one frame at a time. Single-threaded, no auto-inserted buffers. Fully deterministic. | Professional / near-RT workflows — same input always produces same output with same timing. |

The scheduling policy is a pluggable component — the scheduler's dependency
tracking and dispatch logic are policy-agnostic. All policies operate on the
same ready-set; only the selection heuristic differs. New policies can be
added without modifying the scheduler core.

### Blueprint Scheduler and Near-Hard-RT

The blueprint scheduler approaches hard real-time by providing deterministic
execution order, pre-allocated pools, and single-threaded operation. In
practice, event-exceptions (Flushed, FormatChanged, ParamChanged) rarely
occur during steady-state execution in hard-RT use cases:

- **FormatChanged:** normalized by an upstream filter, or avoided entirely.
- **Seeking:** typically not needed in live/broadcast pipelines.
- **ParamChanged:** ill-suited for hard-RT workloads.
- **EndOfStream:** terminates the pipeline — acceptable.

For use cases that need formal hard-RT guarantees beyond what the blueprint
scheduler provides: deterministic memory allocation (all pools pre-sized),
lock-free structures, and page-pinned memory would be required. These can be
layered as optional modes without changing the core design.

## Threading

| Mode | Implementation |
|------|---------------|
| **Single-threaded** | Scheduler loop and all coroutine resumptions on one thread. Deterministic, easy to debug. |
| **Thread pool (N workers)** | Central ready-queue protected by a mutex + condition variable. Workers dequeue and resume. |
| **Work-stealing (N workers)** | Per-worker local deques. Idle workers steal from others. Less contention than a central queue. |

All three modes use the same coroutine interface; only the dispatch backend
changes. Single-threaded mode is always available for testing and debugging,
even when the production configuration is multi-threaded.

The practical path: implement single-threaded first to bootstrap the project,
then graduate to a thread pool. Work-stealing is a later optimization — the
interfaces are the same throughout.

### Device-Affine Thread Groups

The scheduler maintains **per-device thread groups** alongside the general CPU
pool:

- **CPU pool** — general-purpose threads for CPU-bound filters.
- **Device threads** — per-device thread(s) holding the device context (CUDA
  `CUcontext`, Vulkan `VkDevice`, etc.).

A filter's coroutine can migrate between groups mid-execution:

```cpp
co_await ctx.on_device("cuda:0");   // suspend → resume on CUDA:0 thread
// GPU work — running in the correct device context
co_await ctx.on_device("cpu");      // suspend → resume on CPU pool
// CPU post-processing
```

`on_device(dev)` suspends the coroutine and re-enqueues it on the target
device's ready queue. The target thread picks it up on its next dispatch cycle.

This enables:

- **Natural device affinity.** CUDA contexts are thread-bound — running on the
  device thread handles context binding automatically.
- **CPU/GPU overlap.** GPU command submission doesn't block CPU workers.
- **Mixed processing.** A single filter can do CPU prep → GPU compute → CPU
  postprocess without hand-off between separate filters.

Device-affine groups are orthogonal to the choice of scheduling policy and
threading mode. Even the single-threaded scheduler supports `on_device()` by
switching the active device context on the one thread.

### Thread Pinning (Scope Guard)

Some code requires same-thread execution (OpenGL contexts, thread-local
storage, certain hardware APIs). A filter pins itself to the current thread
for a region using a scope guard:

```cpp
{
    auto pin = co_await ctx.pin_thread();   // pinned from here
    gl_bindTexture(...);
    gl_drawArrays(...);
}   // unpinned — scheduler may migrate on next co_await
```

While the guard is active, the scheduler does not migrate the coroutine to a
different worker. Outside the guard, the filter runs on whichever thread the
scheduler picks. This is more flexible than a factory-level hint — a filter
that does CPU prep + GL render + CPU postprocess only needs pinning during
the GL portion.

## Backpressure

Backpressure emerges naturally from the rendezvous link model and coroutine
suspension:

```
  ┌──────────┐                              ┌──────────┐
  │ Producer │──── push() ── rendezvous ── pull() ────│ Consumer │
  └──────────┘     suspends if               suspends if└──────────┘
                   consumer not ready        producer not ready
```

- A link is a rendezvous: the frame passes directly when both sides are ready.
  If the producer pushes before the consumer pulls, the producer suspends. If
  the consumer pulls first, it suspends.
- Where the scheduler auto-inserts `BufferQueue` filters (see below),
  backpressure is bounded by the queue capacity — `push()` suspends when the
  queue is full.

No explicit flow-control messages are needed — the suspension/resumption
mechanism *is* the flow control.

## Auto-Inserted Buffering

Links are rendezvous points with no built-in queue. The scheduler may
auto-insert `BufferQueue` filters in the physical graph where buffering
improves throughput or is needed for correctness. This is a **scheduler-level
decision**, not a core invariant — different scheduler implementations may
choose different strategies.

Typical auto-insertion triggers:

| Situation | Reason |
|-----------|--------|
| **Fan-out branches** | Without buffering, `push()` blocks until the slowest consumer pulls. A buffer on each branch lets the producer continue. |
| **Cross-device links** | Pipeline overlap between CPU and GPU work requires decoupling. (These links already get a transfer filter; a BufferQueue is inserted alongside it.) |
| **Cross-thread links** | If producer and consumer are on different threads, a buffer enables them to run in parallel. |

A minimal-latency scheduler (e.g. the "blueprint" scheduler for deterministic
pipelines) may choose to insert no buffers at all — every link stays a strict
rendezvous.

## Buffer Pool Backpressure

The same mechanism extends to buffer pools. When a pool is exhausted:

- `ctx.alloc()` suspends the coroutine.
- The scheduler resumes it when a buffer is returned to the pool (because a
  downstream filter released a frame).

This creates an end-to-end backpressure chain: source → … → sink. If the sink
is slow, buffers accumulate, pools fill, and upstream filters naturally
throttle.

## Deadlock Detection

Potential deadlocks arise when a cycle of filters are all waiting on each
other. In a DAG, a data-flow deadlock is impossible — there is
always a topological ordering where at least one source or ready filter can
make progress.

Deadlocks *can* still occur from resource starvation (e.g. two filters in
different branches compete for the same fixed-size buffer pool and each
holds one buffer while waiting for another). The scheduler can detect this
by periodically checking for cycles in the wait-for graph. Mitigation
strategies:

- **Timeout:** if a filter has been suspended for longer than a configurable
  threshold, report a diagnostic.
- **Pool partitioning:** assign separate pool budgets per graph branch.
- **Priority donation:** when a pool-waiter is detected, temporarily boost
  the priority of the filter holding the needed buffer.

## GPU Work Integration

A GPU filter typically:

1. Pulls an input frame (CPU-side coroutine operation).
2. Submits GPU work (kernel launch, shader dispatch).
3. Needs to wait for GPU completion before pushing the output.

The "wait for GPU" step is another `co_await`:

```cpp
co_await ctx.device_sync(stream);  // suspends until GPU stream is idle
```

The scheduler integrates with device event/fence mechanisms:

- **CUDA:** `cuStreamAddCallback` or `cuEventQuery` polling.
- **Vulkan:** `vkGetFenceStatus` polling or `vkWaitForFences` on a
  dedicated wait thread.

While a filter is waiting for GPU completion, the CPU thread is free to
execute other ready coroutines — this is where pipeline overlap between
CPU and GPU work happens.
