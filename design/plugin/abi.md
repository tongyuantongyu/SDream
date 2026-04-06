# Plugin ABI

## Entry Point

Every plugin shared library exports a single C function:

```c
SDREAM_EXPORT int32_t sdream_plugin_register(
    SDreamPluginRegistrar* registrar,
    uint32_t               host_abi_version
);
```

- `registrar` — opaque handle; the plugin calls methods on it to register
  extensions.
- `host_abi_version` — the ABI version the framework speaks. The plugin
  should check this and return `SDREAM_STATUS_UNSUPPORTED` if it cannot
  work with this version.
- Returns `SDREAM_STATUS_OK` (0) on success, negative on failure.

## ABI Versioning

The ABI version is a single `uint32_t` encoding `major.minor`:

```
version = (major << 16) | minor
```

- **Major bump:** breaking changes (struct layout change, removed function).
  Plugin must refuse to load if the host major differs from its own.
- **Minor bump:** backward-compatible additions (new registrar methods, new
  optional struct fields appended at the end). A plugin compiled against
  minor N works with host minor ≥ N.

Struct extensibility: all ABI structs carry a `struct_size` field as their
first member. The host and plugin can detect version skew by comparing
`struct_size` against the expected size, and safely ignore trailing fields
they don't understand.

```c
typedef struct SDreamFilterFactory {
    uint32_t    struct_size;       // = sizeof(SDreamFilterFactory)
    const char* name;
    // … fields …
} SDreamFilterFactory;
```

## Plugin Registrar

The registrar provides these callback functions:

```c
typedef struct SDreamPluginRegistrar {
    void (*register_filter_factory)(
        SDreamPluginRegistrar*,
        const SDreamFilterFactory*);
    void (*register_device_provider)(
        SDreamPluginRegistrar*,
        const SDreamDeviceProvider*);
    void (*register_auto_converter)(
        SDreamPluginRegistrar*,
        const SDreamAutoConverter*);
    void (*register_transfer_bridge)(
        SDreamPluginRegistrar*,
        const SDreamTransferBridge*);
} SDreamPluginRegistrar;
```

## Filter Factory (C ABI)

```c
typedef struct SDreamFilterFactory {
    uint32_t    struct_size;

    const char* name;              // unique identifier, e.g. "sdream.video.scale"
    const char* display_name;      // human-readable, e.g. "Video Scaler"
    const char* category;          // hierarchical, e.g. "video/transform"
    const char* description;

    // Pin templates
    uint32_t              n_input_pins;
    SDreamPinTemplate*    input_pins;
    uint32_t              n_output_pins;
    SDreamPinTemplate*    output_pins;

    // Device affinity (preference order)
    uint32_t              n_supported_devices;
    SDreamFourCC*         supported_devices;

    // Parameter schema — see [D13] in decisions.md
    uint32_t              n_params;
    SDreamParamDesc*      params;

    // Lifecycle
    SDreamFilterCreateFn  create;       // allocate filter state → void*
    SDreamFilterDestroyFn destroy;      // free filter state
    SDreamFilterStepFn    step;         // coroutine step function (see cross-language.md)
} SDreamFilterFactory;
```

### Pin Template

```c
typedef struct SDreamPinTemplate {
    const char*     name;           // e.g. "video", "audio_0"
    SDreamDirection direction;      // SDREAM_DIR_INPUT or SDREAM_DIR_OUTPUT
    SDreamPresence  presence;       // ALWAYS, SOMETIMES, REQUEST
    uint32_t        n_caps;
    SDreamCapability* caps;         // array of capability constraints
} SDreamPinTemplate;
```

### Parameter Descriptor — schema format [D13] pending

```c
typedef enum {
    SDREAM_PARAM_INT,
    SDREAM_PARAM_FLOAT,
    SDREAM_PARAM_STRING,
    SDREAM_PARAM_BOOL,
    SDREAM_PARAM_ENUM,
    SDREAM_PARAM_RATIONAL,
} SDreamParamType;

typedef struct SDreamParamDesc {
    const char*     key;            // e.g. "width"
    const char*     display_name;   // e.g. "Output Width"
    const char*     description;
    SDreamParamType type;
    SDreamValue     default_value;

    // Validation (type-dependent)
    SDreamValue     min_value;      // for INT, FLOAT, RATIONAL
    SDreamValue     max_value;
    uint32_t        n_enum_values;  // for ENUM
    const char**    enum_labels;
    const int32_t*  enum_values;

    uint32_t        flags;          // SDREAM_PARAM_READONLY, SDREAM_PARAM_DYNAMIC, …
} SDreamParamDesc;
```

`SDREAM_PARAM_DYNAMIC` indicates the parameter can be changed at runtime
Parameters with this flag are delivered to the filter via `ParamChanged`
exception at the next suspension point. Non-dynamic parameters are fixed
after `prepare()`.

## Plugin Discovery

Plugins are located by (in priority order):

1. **Explicit registration** — the application calls
   `sdream_register_plugin_path(path)` before building the graph.
2. **Directory scan** — the framework scans paths in `$SDREAM_PLUGIN_PATH`
   (or a compile-time default) for shared libraries matching the naming
   convention `sdream_plugin_*{.dll,.so,.dylib}`.
3. **Manifest file** — an optional JSON file listing plugin paths and their
   expected registrations, allowing lazy loading (only load when a filter
   name is requested).

Duplicate registrations (same filter name from multiple plugins) are resolved
by priority order above; later registrations shadow earlier ones, with a
warning logged.
