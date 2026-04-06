# Error Handling

## Design Principle

Errors are values, not exceptions, at every boundary:

- The **C ABI** uses `Status` codes.
- The **C++ internals** use whichever style (`std::expected` or exceptions) is
  cleanest per-module. Either way, errors are caught and converted to `Status`
  at the ABI boundary.
- Within the **graph**, errors are either returned from the coroutine (fatal)
  or reported as non-fatal events.

## Status Codes

```cpp
enum class Status : int32_t {
    Ok              =  0,
    EndOfStream     =  1,   // not an error — normal termination signal
    TryAgain        =  2,   // transient: resource temporarily unavailable

    // Errors (negative)
    InvalidArgument = -1,
    Unsupported     = -2,
    OutOfMemory     = -3,
    DeviceLost      = -4,   // GPU reset, device removed
    IoError         = -5,
    Cancelled       = -6,
    Timeout         = -7,
    CorruptData     = -8,
    NegotiationFailed = -9,
    InternalError   = -100,
};
```

Positive values are non-error conditions. Negative values are errors. Ranges
are reserved for extension.

## Error Categories

| Category | Examples | Handling |
|----------|----------|----------|
| **Transient** | Pool exhausted, network timeout | Framework retries or suspends and retries later. Filter may see `TryAgain`. |
| **Data corruption** | Bitstream error, CRC mismatch | Filter calls `ctx.report_error(CorruptData)`. Framework logs it. Downstream may receive a frame with `Corrupt` flag. Stream continues. |
| **Fatal (filter)** | Codec init failure, device lost | Filter's coroutine returns an error status. Framework applies a recovery policy (see below). |
| **Fatal (framework)** | Out of memory in scheduler, internal assertion | Framework tears down the graph and reports to the application. |

## Error Propagation in the Graph

### Non-Fatal (Data-Level)

```
Filter A detects corruption
  → calls ctx.report_error(CorruptData, "CRC mismatch at offset 0x…")
  → framework logs the error
  → filter produces a best-effort frame with Corrupt flag
  → downstream filters see the flag and can:
      - Process normally (ignore corruption)
      - Skip the frame
      - Replace with a placeholder (e.g. last good frame)
```

### Fatal (Filter-Level)

```
Filter A encounters an unrecoverable error
  → coroutine returns Status::DeviceLost (or throws, caught at boundary)
  → framework transitions filter to Error state
  → framework applies recovery policy:

  Policy 1 — Teardown (default):
    → Send Error event downstream.
    → Drain remaining filters.
    → Graph transitions to Error state.
    → Application receives the error.

  Policy 2 — Isolate:
    → Disconnect the failed filter's subgraph.
    → Other independent subgraphs continue.
    → Application is notified.

  Policy 3 — Restart:
    → Destroy the failed filter.
    → Create a new instance from the same factory.
    → Re-negotiate.
    → Resume (may lose some frames).
```

The recovery policy is configurable per-graph or per-filter.

## Diagnostic Context

Every error carries optional diagnostic context:

```cpp
struct ErrorContext {
    Status      code;
    const char* message;         // human-readable
    const char* filter_name;     // which filter originated the error
    const char* source_location; // file:line in filter code (debug builds)
};
```

The framework's logger receives these and the application can register an
error callback for programmatic handling.

## ABI Boundary Safety

At the C ABI boundary, the framework wraps every call to a filter's step
function:

```
status = filter->step(state, &response, &request);
if (status < 0) {
    // filter reported an error — enter error handling path
}
```

If the C++ convenience SDK is used, exceptions thrown inside the SDK wrapper
are caught and converted to `Status` codes before crossing the boundary. The
C ABI itself never exposes exceptions.
