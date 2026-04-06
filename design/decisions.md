# Design Decisions

Trade-offs where multiple valid approaches exist. Each decision is
independent. Fill in the **Your Choice** column.

---

## D01. Graph Topology — DAG Only or Allow Cycles?

Cycles enable feedback loops (audio echo, video feedback, iterative
refinement).

| | **DAG only** | **Allow cycles** |
|---|---|---|
| Complexity | Simple — topological sort gives a static execution order | Back-edges need mandatory initial buffering to break deadlocks |
| Deadlock | Impossible by construction | Must be prevented: every cycle requires ≥1 pre-filled frame |
| Scheduling | Straightforward | Scheduler must handle circular dependencies |
| Use cases lost | Feedback effects, iterative algorithms | None |

**Recommendation:** start DAG-only. Design the `Link` abstraction so that a
"back-edge with seed frames" can be added later without core redesign.

Question: are these better implemented as graph structure or intra-filter behavior? What are the examples that really need cycle in graph? 

---

## D02. Scheduling Strategy

How the scheduler selects which suspended filter to resume next.

| | **Demand-driven (pull)** | **Supply-driven (push)** | **Hybrid (level-based)** |
|---|---|---|---|
| Concept | Start from sinks; wake producers whose output queues are below threshold | Start from sources; wake consumers whose input queues have data | Monitor all queues; resume whichever filter can make the most progress |
| Latency | Higher (frames produced only on demand) | Lower (frames produced eagerly) | Tunable |
| Peak memory | Lower (only produce what's needed) | Higher (sources run ahead) | Bounded by queue caps either way |
| Batch / offline | Excellent | Acceptable | Good |
| Live / streaming | Acceptable | Excellent | Good |
| Complexity | Low | Low | Medium |

**Recommendation:** hybrid is the most flexible — the scheduler already knows
each filter's blocking reason because every suspension goes through it. A
demand-driven mode is a good first milestone since it naturally bounds memory
and is simpler to prove correct.

Answer: The beauty of using coroutine is that filter can be scheduled in eitehr way, while not creating extra burden on filter author. We can have multiple schedulers tuned for different use cases.

---

## D03. Threading & Concurrency Model

| | **Single-threaded cooperative** | **Fixed thread pool** | **Work-stealing pool** |
|---|---|---|---|
| Concept | One OS thread; coroutines yield cooperatively | N threads; central ready-queue dispatches to workers | N threads with per-thread local queues; idle threads steal |
| Multi-core usage | None | Good | Excellent |
| Sync overhead | Zero | Moderate (shared ready-queue contention) | Low (local queues; steal only when idle) |
| GPU integration | Works (CPU thread submits and awaits GPU completion) | Each worker can submit GPU work | Same, with better utilization |
| Determinism | Full (useful for tests, debugging) | Non-deterministic | Non-deterministic |
| Effort | Trivial | Moderate | High |

**Notes:**
- Single-threaded mode is valuable for reproducible tests and debugging
  regardless of the production choice.
- The coroutine suspension model is identical across all options; only the
  scheduler's dispatch logic changes.
- A practical path: implement single-threaded first, then graduate to a thread
  pool. Work-stealing can be added as an optimization later — the interfaces
  are the same.

Answer: we are not limited to having only one scheduler. Practically we can implement the simplest first, and gradually add / replace with the more complicated ones.

Would it better to have device-dedicated threads? With coroutine, filters can simply (example, not final API)

```c++
co_await sdream::to_device("cuda:0");
co_await sdream::to_device("cpu");
```

to teleport between devices.

---

## D04. Buffer Ownership Semantics

| | **Unique (move)** | **Shared (ref-counted)** | **Shared + Copy-on-Write** |
|---|---|---|---|
| Concept | Each buffer has exactly one owner; transfer = move | Multiple references to same buffer; last release frees | Shared read; any write triggers a private copy |
| Zero-copy fan-out | No — must copy for each downstream | Yes | Yes |
| In-place mutation | Always safe (exclusive) | Unsafe — writer must ensure sole ownership | Safe — auto-copies when ref_count > 1 |
| Overhead | No atomics | Atomic ref-count inc/dec | Ref-count + branch on every write |
| Complexity | Low | Low | Medium |

**Recommendation:** shared (ref-counted) is the industry standard for media
frameworks. Copy-on-write adds safety at the cost of a branch on every write
path; it could be offered as an opt-in wrapper rather than baked into the core.

A filter that wants to mutate a frame it pulled can call
`ctx.make_writable(frame)`, which is a no-op when ref_count == 1 and copies
otherwise. This keeps the fast path fast.

Answer: Recommendation and filter opt-in sounds good.

---

## D05. Format Negotiation — Static vs Dynamic

| | **Static only** | **Static + mid-stream renegotiation** |
|---|---|---|
| Concept | Formats fixed at `prepare()` time | Formats may change mid-stream via `FormatChanged` event |
| Complexity | Low | High — every filter must handle re-init on format change |
| Resolution/rate changes | Require graph rebuild | Handled in-place |
| Adaptive streaming | Not supported | Supported |
| Typical need | Offline transcoding, batch | Live playback, adaptive bitrate |

**Recommendation:** even under static-only, reserve the `FormatChanged` event
code so the door stays open. The negotiation infrastructure is the same either
way; the difference is whether filters' running states must be re-entrant.

Answer: The key here is to keep the filter code clean. Consider an typical filter with our design:

```c++

ReturnType filter(Ctx ctx) {
  // setup
  // auto buffer = co_await ctx.allocate();


  while (true) {
    auto frame = co_await ctx.frame();
    if (!frame) {
      return;
    }

    // process
  }
}
```

How to communicate the `FormatChanged` event?

I'm considering making it an exception: `FormatChanged`-aware filters catch and handles it, while unaware filters simply propagate and teardown. Framework notices the bubbled exception and recreate the filter. Do you see any concern?

I'm also considering allow filter to expect more thing to not change, like fps. And some filters may be fully intra and basically don't care, they can suppress the event.

---

## D06. Plugin ABI Surface

| | **Pure C ABI** | **C ABI + C++ convenience SDK** |
|---|---|---|
| Concept | Everything is C structs and function pointers | C ABI at the shared-library boundary; a header-only C++ SDK wraps it with RAII, templates, strong types |
| FFI to other languages | Trivial | C++ layer is unusable from FFI; C layer still is |
| C++ developer ergonomics | Verbose | Pleasant |
| ABI stability | Stable (C rules) | C boundary stable; C++ SDK is header-only ⇒ no binary ABI concern |

**Recommendation:** C ABI boundary + C++ convenience SDK. This is nearly
universal in successful cross-language plugin systems.

Answer: Is it worth it to expose C++ ABI, so C++ filters built with compatible compiler toolset don't need C translation?

Ensure that in any supported language, the user only need to define a coroutine, and a few lines of registration code.  

---

## D07. Internal Error Handling Style (C++ Side)

The C ABI boundary always uses `Status` codes. This decision is about the
internal C++ code and the C++ convenience SDK.

| | **Exceptions** | **`std::expected<T,E>` / Result types** |
|---|---|---|
| Happy-path ergonomics | Clean — errors are invisible until they happen | Every call site handles or propagates explicitly |
| Performance | Zero-cost happy path (table-based EH); costly on throw | Tiny branch on every call |
| Coroutine interaction | Works, but exceptions inside coroutine frames have subtle lifetime issues | `co_await` naturally returns `expected<T,E>` — no special cases |
| C ABI mapping | Must catch at boundary and convert | Maps 1:1 to C status codes |

**Notes:** the two styles can coexist — `expected` at the coroutine/scheduler
level, exceptions in standalone utility code. But a single dominant convention
avoids confusion.

---

## D08. Seek Model

| | **Flush-seek** | **Segment-based (GStreamer-style)** |
|---|---|---|
| Concept | Flush all queues, seek source, restart pipeline | Source sends new `Segment` event; filters rebase timestamps without flushing |
| Seek latency | Higher (flush + refill) | Lower (no flush for contiguous segments) |
| Gap-less looping | Difficult | Natural |
| Trick modes (fast-forward, rewind) | Flush on every speed change | New segment with changed rate |
| Complexity | Low | High (segment accounting in every filter) |

**Recommendation:** flush-seek covers the majority of use cases and is a solid
MVP. Segment-based can be layered on top later — the event infrastructure is
the same; the difference is filter-level segment bookkeeping.

---

## D09. Timestamp Representation

| | **Absolute (fixed time base)** | **Per-link time base** | **Segment-relative** |
|---|---|---|---|
| Concept | All timestamps in one unit (e.g. nanoseconds, 1/90000 s) | Each link negotiates its own `Rational` time base (like FFmpeg) | Timestamps relative to current segment start |
| Simplicity | Highest | Medium | Lowest |
| Precision | May overflow for long streams at high resolution | Each stream picks optimal base | Same overflow risk as absolute |
| Seeking | Must rebase manually | Must rebase manually | Segment boundary resets base naturally |
| Conversion cost | None | On every cross-link comparison | On every cross-segment comparison |

**Notes:** absolute nanoseconds is the simplest and covers practical stream
lengths (int64 nanoseconds overflows after ~292 years). Per-link time base
avoids rounding errors for frame-rate-exact timestamps (e.g. 1/24000 for
23.976 fps). The two can be combined: store ticks + time_base per frame, but
adopt a canonical base for cross-link comparisons.

---

## D10. Graph Dynamism

| | **Static** | **Parameter-only** | **Full dynamic** |
|---|---|---|---|
| Concept | Topology fixed after `prepare()` | Topology fixed; filter parameters adjustable at runtime via messages | Filters/links added and removed while graph is running |
| Complexity | Low | Low–Medium | High (partial drain, lock coordination, reconnection) |
| Use cases | Offline transcoding, batch | Live color grading, EQ, gain | Live production switchers, adaptive pipelines |
| Scheduler impact | None | Minimal (param change = message to filter) | Major (partial graph teardown/buildup) |

**Recommendation:** parameter-only is the sweet spot for a first version.
Full dynamic reconfiguration is powerful but imposes significant complexity on
the scheduler and on every filter's lifecycle.

---

## D11. Inter-Filter Queue Sizing

| | **Fixed global** | **Per-link configurable** | **Adaptive** |
|---|---|---|---|
| Concept | All queues same capacity (e.g. 4 frames) | Each link's capacity is configurable (with a default) | Framework adjusts sizes at runtime based on throughput |
| Memory predictability | High | High | Low |
| Tuning burden | None (but may be wrong) | On user/integrator | None (self-tuning) |
| Complexity | Trivial | Low | Medium–High |

**Recommendation:** per-link configurable with sensible defaults. Adaptive
is a later optimization — its heuristics are hard to get right without real
workload data.

---

## D12. FFmpeg Integration Depth

The framework needs codecs, demuxers, and muxers. Reimplementing these is a
multi-year effort.

| | **Wrap FFmpeg** | **Native only** | **Hybrid** |
|---|---|---|---|
| Concept | SDream filter wrappers around libavcodec / libavformat | Rewrite from scratch | FFmpeg as an optional backend; native filters where beneficial |
| Time to useful system | Fast | Years | Medium |
| Dependency weight | Heavy | None | Optional |
| License | LGPL/GPL depending on build | Clean | FFmpeg portion is LGPL/GPL |
| Control over threading/memory | Limited by FFmpeg internals | Full | Per-component choice |

**Recommendation:** hybrid. Wrap FFmpeg for breadth; ensure filter interfaces
are clean enough that native implementations can be dropped in as replacements
over time.

---

## D13. Configuration & Parameter System

| | **Stringly-typed** | **Schema-typed** |
|---|---|---|
| Concept | Parameters are string key–value pairs, parsed by each filter | Each parameter has a declared type, default, valid range, and description |
| Validation | At filter init (late, poor diagnostics) | At graph construction (early, precise) |
| Introspection / GUI generation | Difficult | Straightforward |
| Effort | Low | Medium |

**Recommendation:** schema-typed. The upfront effort pays off in better error
messages, auto-generated documentation, and GUI support.

---

## D14. In-Place Processing Fast Path

Some filters can process data in-place (gain adjustment, color LUT, overlay
burn-in). An explicit in-place path avoids a buffer allocation + copy.

| | **Always allocate output** | **In-place when possible** |
|---|---|---|
| Concept | Every filter allocates a fresh output buffer via `ctx.alloc()` | Filter declares `supports_inplace`; framework provides a writable view of the input buffer when ref_count == 1 |
| Memory traffic | Higher (read input + write output) | Lower (modify in place) |
| Complexity | Uniform — one code path | Filter must handle both paths; framework must track mutability |
| Safety | High (input never mutated) | Requires discipline (only when sole owner) |

**Notes:** this is an optimization, not a fundamental design choice. A
`ctx.make_writable(frame)` helper (which copies only when shared) provides
the same benefit without a separate in-place mode. The question is whether
the framework should *automatically* offer in-place when it detects sole
ownership, or leave it to the filter author to call `make_writable`.

---

## Summary

| ID | Topic | Options | Your Choice |
|----|-------|---------|-------------|
| D01 | Graph topology | DAG only / Allow cycles | |
| D02 | Scheduling strategy | Demand-driven / Supply-driven / Hybrid | |
| D03 | Threading model | Single-threaded / Thread pool / Work-stealing | |
| D04 | Buffer ownership | Unique / Shared / Shared+CoW | |
| D05 | Format negotiation | Static only / Static + renegotiation | |
| D06 | Plugin ABI | Pure C / C ABI + C++ SDK | |
| D07 | Error handling (C++) | Exceptions / Result types / Mixed | |
| D08 | Seek model | Flush-seek / Segment-based | |
| D09 | Timestamp representation | Absolute / Per-link time base / Segment-relative | |
| D10 | Graph dynamism | Static / Parameter-only / Full dynamic | |
| D11 | Queue sizing | Fixed / Per-link configurable / Adaptive | |
| D12 | FFmpeg integration | Wrap FFmpeg / Native / Hybrid | |
| D13 | Parameter system | Stringly-typed / Schema-typed | |
| D14 | In-place processing | Always alloc / In-place when possible | |
