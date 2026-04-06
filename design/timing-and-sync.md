# Timing & Synchronization

## Pipeline Clock

A graph has a single **pipeline clock** — a monotonically increasing time
reference used by sinks to pace output.

### Clock Types

| Type | Source | Use case |
|------|--------|----------|
| **System** | `std::chrono::steady_clock` | General real-time playback. |
| **Audio** | Audio device callback timing (e.g. WASAPI, ALSA period interrupts). | A/V playback — smoothest when audio is the master. |
| **Freerun** | Advances as fast as the pipeline can produce frames. | Offline transcoding, benchmarking. |
| **External** | NTP, PTP, genlock signal. | Broadcast, multi-machine sync. |

The default is `System`. The application selects a clock before `run()`.

### Clock Selection

The audio clock is preferred for playback because audio output is the most
latency-sensitive path — small jitter in audio is immediately perceptible,
while a skipped or repeated video frame is barely noticeable. When an audio
sink is present in the graph, it can promote itself to clock master.

Under the freerun clock, sinks never wait — they consume frames as fast as
they arrive. This maximizes throughput for batch processing.

---

## A/V Synchronization

Video sinks compare each frame's PTS against the pipeline clock:

```
offset = frame.pts - clock.now()

if offset > +threshold:
    wait(offset)                  // frame is early — sleep
elif offset >= -drop_threshold:
    present(frame)                // on time or slightly late — show it
else:
    drop(frame)                   // too late — skip
    send QoS upstream
```

### Thresholds

| Parameter | Typical default | Meaning |
|-----------|----------------|---------|
| `threshold` | half a frame duration | How early to wake up for presentation. |
| `drop_threshold` | one frame duration | How late a frame can be before it's dropped. |

These are configurable per-sink.

---

## Quality of Service (QoS)

When a sink drops a frame, it sends a `QoS` event upstream:

```
QoS {
    jitter:     Timestamp,   // how late the frame was (negative = late)
    proportion: float,       // ratio of on-time frames recently (0.0–1.0)
    timestamp:  Timestamp,   // PTS of the problematic frame
}
```

Upstream filters can react:

| Filter type | Possible QoS response |
|-------------|----------------------|
| **Decoder** | Skip non-reference frames (B-frames). Decode at lower quality. |
| **Scaler / effects** | Switch to a faster (lower quality) algorithm. |
| **Source** | Increase read-ahead buffer size. |

QoS responses are optional — a filter that does not handle QoS simply forwards
the event upstream.

---

## Multi-Stream Synchronization

When a graph has parallel paths (video + audio + subtitles), alignment is
needed at convergence points (muxers, renderers):

1. Each stream carries its own PTS in its own time base.
2. The convergence filter (e.g. muxer) converts all timestamps to a common
   base for comparison.
3. Streams are interleaved in PTS order. If one stream runs ahead, the muxer
   holds its frames until the other streams catch up (bounded by a
   configurable max-interleave duration to avoid unbounded buffering).

For a renderer (e.g. video + subtitle overlay), the subtitle stream's PTS
determines visibility windows — the renderer shows/hides subtitle events based
on the video frame's PTS.

---

## Latency Management

### Latency Accumulation

Each filter reports its inherent processing latency — the minimum time between
receiving an input frame and producing the corresponding output. Examples:

| Filter | Latency | Reason |
|--------|---------|--------|
| Passthrough | 0 | Immediate. |
| Video encoder | N × frame_duration | B-frame reordering + lookahead. |
| Audio resampler | chunk_size / sample_rate | Internal buffering. |
| Device transfer | DMA time | Hardware-dependent. |

The framework sums latency along each source-to-sink path:

```
total_latency(path) = Σ filter_latency(f) for f in path
```

The application can query this to adjust initial buffering (e.g. pre-roll
enough audio to cover the video path's latency).

### Latency Events

A `Latency` upstream event propagates from sinks to sources:

1. Sink starts with its own latency (e.g. audio device buffer size).
2. Each filter adds its own latency and forwards upstream.
3. Source receives the total downstream latency and can adjust read-ahead.

### Low-Latency Mode

For live capture / conferencing / monitoring:

- Default queue sizes reduced to 1–2 frames.
- Buffer pool capacities minimized.
- Encoder pre-fill / lookahead disabled.
- QoS thresholds tightened.

This is a graph-level flag that overrides per-link defaults.
