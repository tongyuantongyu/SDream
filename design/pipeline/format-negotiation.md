# Format Negotiation

## Purpose

Before data flows, every link needs a concrete `MediaType` — both sides must
agree on resolution, pixel format, sample rate, etc. Negotiation resolves this
automatically, inserting converters when needed.

## When It Runs

Negotiation happens during `graph.prepare()`, after all filters are
instantiated and linked but before the scheduler starts. It is a compile-time
step for the pipeline — not a runtime cost.

## Algorithm

```
1. Collect capabilities.
   For each filter:
     - Output pins publish their capability sets (what they can produce).
     - Input pins publish their capability sets (what they can accept).
   Some filters derive output caps from input caps (e.g. a scaler produces
   any resolution but the same pixel format as its input). These are
   modeled as "depends on input" constraints.

2. Intersect per link.
   For each link (upstream out_pin → downstream in_pin):
     intersection = intersect(out_pin.caps, in_pin.caps)

3. Handle empty intersections.
   If intersection is empty:
     a. Try inserting an auto-converter filter between the two pins.
        Common auto-converters:
          - Pixel format converter (e.g. NV12 → I420)
          - Audio resampler (e.g. 44100 → 48000)
          - Channel remapper
        Re-run intersection with the converter's caps bridging the gap.
     b. If no converter can bridge the gap → fail with a diagnostic:
        "Cannot connect filter A output (NV12, CUDA) to filter B input
         (RGB24, CPU): no compatible format and no auto-converter available."

4. Select concrete format.
   From the non-empty intersection, pick a concrete MediaType using a
   preference heuristic:
     - Prefer the upstream filter's native format (avoid unnecessary work).
     - Prefer wider formats over narrower (e.g. 10-bit over 8-bit if both
       sides support it) to avoid quality loss.
     - Prefer the format that minimizes downstream conversions.
   The heuristic is best-effort; users can override by setting explicit
   format constraints on a link.

5. Propagate.
   Fixating a link's format may constrain dependent pins. For example,
   fixing a scaler's input to NV12 may narrow its output caps.
   Re-run intersection for affected links.

6. Repeat until fixed point.
   If no more links change → negotiation is complete.
   If a contradiction is found (a link's intersection becomes empty after
   propagation) → fail with a diagnostic.
```

### Complexity

In practice, multimedia graphs are small (tens to low hundreds of filters) and
each filter has few pins. The fixed-point loop converges quickly — typically
1–3 passes. Worst case is O(L × C) where L = number of links and C = average
capability set size.

## Auto-Converter Registry

The framework maintains a registry of auto-converter factories, each
describing:

- Input capability set.
- Output capability set.
- Cost estimate (used when multiple converters could bridge a gap — prefer the
  cheapest).

Plugins can register additional auto-converters, for example:

- A GPU color-space converter that can run on CUDA.
- A hardware-accelerated resampler.

## User Overrides

Users can constrain a link's format explicitly:

```cpp
g.link(decode, "out", scale, "in", {{"pixel_format", "NV12"}});
```

This narrows the capability intersection before the algorithm runs. If the
constraint makes negotiation impossible, the error message includes the
user-specified constraint for clarity.

## Mid-Stream Renegotiation

When a format change occurs at runtime (e.g. adaptive bitrate switch,
resolution change from an upstream source), the framework delivers a
`FormatChanged` exception to downstream filters. The handling depends on each
filter's declaration:

1. **Format-resilient filters** — exception suppressed. They process the next
   frame with the new format naturally.
2. **Default filters that catch `FormatChanged`** — reinitialize internal
   state for the new format and continue. May throw a fatal error if the
   specific change is incompatible (e.g. resolution change mid-interpolation).
3. **Default filters that don't catch** — the exception propagates; the
   framework destroys and recreates the filter with the new format.

After all affected filters have handled or been recreated, the framework
updates the link MediaTypes and resumes data flow. Auto-inserted converters
are re-evaluated and replaced if the new format requires a different
conversion path.

See [Stream Events](stream-events.md#event-exceptions) for the full
event-exception model.
