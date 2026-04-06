# Cross-Language Support

## The Problem

C++20 coroutines are a compiler feature — they cannot be expressed through a
C ABI. Filters written in other languages need a way to participate in the
same coroutine scheduling model without using C++ coroutines directly.

## Solution: State-Machine Step Protocol

The filter's coroutine is decomposed into an explicit state machine. The
framework calls a **step function** repeatedly; each call advances the filter's
state by one suspension point.

```
┌───────────────────────────────────────────────────┐
│              Framework (scheduler)                 │
│                                                    │
│  ┌────────┐   response   ┌────────────┐           │
│  │ step() │──────────────▶│   Filter   │           │
│  │        │◀──────────────│   (any     │           │
│  │        │   request     │   language)│           │
│  └────────┘               └────────────┘           │
│       │                                            │
│  framework fulfills the request (pull, alloc, …)   │
│  then calls step() again with the response         │
└───────────────────────────────────────────────────┘
```

### Step Function Signature

```c
typedef enum {
    SDREAM_ACTION_NEGOTIATE = 0,  // "I need negotiation results"
    SDREAM_ACTION_PULL      = 1,  // "I need a frame from input pin N"
    SDREAM_ACTION_PUSH      = 2,  // "I'm pushing a frame to output pin N"
    SDREAM_ACTION_ALLOC     = 3,  // "I need a buffer of MediaType T"
    SDREAM_ACTION_SYNC      = 4,  // "Wait for device stream completion"
    SDREAM_ACTION_DONE      = 5,  // "I'm finished (EOS)"
    SDREAM_ACTION_ERROR     = 6,  // "I've failed"
} SDreamAction;

typedef struct SDreamStepIO {
    SDreamAction     action;
    int32_t          pin_index;      // for PULL / PUSH
    SDreamFrame*     frame;          // for PUSH: frame to send
                                     // for PULL response: received frame
    SDreamMediaType* media_type;     // for ALLOC
    SDreamDeviceStreamHandle stream; // for SYNC
    int32_t          error_code;     // for ERROR
    const char*      error_message;  // for ERROR
} SDreamStepIO;

typedef int32_t (*SDreamFilterStepFn)(
    void*                filter_state,   // opaque — allocated by create()
    const SDreamStepIO*  response,       // framework's answer to previous request
                                         // NULL on first call
    SDreamStepIO*        request         // filter fills this with its next need
);
```

### Protocol Flow

```
Framework calls: step(state, NULL, &req)            // initial call
  Filter returns: req = { action: NEGOTIATE }

Framework runs negotiation, then calls:
  step(state, &{negotiated_formats}, &req)
  Filter returns: req = { action: PULL, pin: 0 }

Framework waits for data, then calls:
  step(state, &{frame: input_frame}, &req)
  Filter processes, returns: req = { action: ALLOC, media_type: out_fmt }

Framework allocates, then calls:
  step(state, &{frame: empty_output_buffer}, &req)
  Filter fills buffer, returns: req = { action: PUSH, pin: 0, frame: output }

… and so on until filter returns DONE or ERROR.
```

### Framework-Side Adapter

The framework wraps the step function in an internal C++ coroutine so that the
scheduler treats it identically to a native C++ filter coroutine:

```cpp
Task step_adapter(void* state, SDreamFilterStepFn step, FilterContext& ctx) {
    SDreamStepIO request, response;
    int32_t status = step(state, nullptr, &request);
    while (status >= 0 && request.action != SDREAM_ACTION_DONE) {
        switch (request.action) {
        case SDREAM_ACTION_PULL:
            response.frame = (co_await ctx.pull(request.pin_index)).release();
            break;
        case SDREAM_ACTION_PUSH:
            co_await ctx.push(request.pin_index, Frame::from_c(request.frame));
            break;
        case SDREAM_ACTION_ALLOC:
            response.frame = (co_await ctx.alloc(
                MediaType::from_c(request.media_type))).release();
            break;
        // … other actions …
        }
        status = step(state, &response, &request);
    }
}
```

This adapter is zero-overhead beyond the function-pointer call — there is no
boxing, serialization, or thread boundary.

---

## Language Scaffolding

### C

A set of macros manages the state enum and the switch statement:

```c
SDREAM_FILTER_BEGIN(my_filter)
    SDREAM_NEGOTIATE(fmts)
    // setup code using fmts …

    while (1) {
        SDREAM_PULL(0, in_frame)
        if (sdream_frame_is_eos(in_frame)) break;

        SDREAM_ALLOC(out_fmt, out_frame)
        process(in_frame, out_frame);

        SDREAM_PUSH(0, out_frame)
    }
SDREAM_FILTER_END
```

The macros expand to a switch-on-state with `case` labels at each suspension
point (Duff's device pattern). The developer writes straight-line code;
the macros handle the state machine.

### Rust

A proc-macro transforms an async function into a C-ABI step function:

```rust
#[sdream::filter]
async fn my_filter(ctx: &mut FilterContext) -> Result<()> {
    let fmts = ctx.negotiate().await;

    loop {
        let input = ctx.pull(0).await;
        if input.is_eos() { break; }

        let mut output = ctx.alloc(&fmts.output(0)).await;
        process(&input, &mut output);

        ctx.push(0, output).await;
    }
    Ok(())
}
```

The proc-macro generates:
- A `#[no_mangle] extern "C"` step function.
- A state struct holding all live locals across await points.
- A create/destroy pair for the state struct.

### Python

A bridge lets the developer write a generator:

```python
@sdream.filter(
    inputs=[sdream.Pin("in", caps=[sdream.video_any()])],
    outputs=[sdream.Pin("out", caps=[sdream.video_any()])],
)
def my_filter(ctx):
    fmts = yield sdream.Negotiate()

    while True:
        frame = yield sdream.Pull(0)
        if frame.is_eos():
            break

        out = yield sdream.Alloc(fmts.output(0))
        process(frame, out)

        yield sdream.Push(0, out)
```

Each `yield` suspends the generator. The SDream Python bridge translates
yields into `SDreamStepIO` requests and feeds responses back via
`generator.send()`.

**Performance note:** Python filters are CPU-bound on the GIL. They are
practical for light-weight logic (metadata routing, subtitle processing) and
for prototyping. For performance-critical paths, use C/C++/Rust.

### C#

A source generator maps `async`/`await`:

```csharp
[SDreamFilter("my_filter")]
public static async Task Run(FilterContext ctx) {
    var fmts = await ctx.Negotiate();

    while (true) {
        var input = await ctx.Pull(0);
        if (input.IsEos) break;

        var output = await ctx.Alloc(fmts.Output(0));
        Process(input, output);

        await ctx.Push(0, output);
    }
}
```

The source generator produces a native step function via P/Invoke marshalling,
with the C# async state machine serving as the filter state.
