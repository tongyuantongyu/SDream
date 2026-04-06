# Execution Model

This section describes how filters run — the runtime behavior of the
framework.

SDream's execution model is built on a single idea: **a filter body is a
coroutine, and every potentially-waiting operation is a suspension point
managed by the framework's scheduler.**

This design collapses the traditional distinction between push-based and
pull-based pipelines. A filter can `co_await pull()` input (pull semantics)
and `co_await push()` output (push semantics) in the same body. The scheduler
decides execution order based on resource availability, not a hardwired
push/pull convention.

## Sub-Pages

| Document | Contents |
|----------|----------|
| [Coroutine Model](coroutine-model.md) | Filter body, async API surface, C++20 mapping, examples |
| [Scheduler](scheduler.md) | Scheduling loop, threading, backpressure, deadlock, GPU integration |
