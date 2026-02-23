# Query.Farm Telemetry Client

A lightweight telemetry client for [DuckDB](https://duckdb.org/) C++ extensions that reports basic platform and version information to [Query.Farm](https://query.farm). This is the canonical source for the telemetry integration used across Query.Farm extensions. Copy the two source files into your extension and call a single function — no external dependencies required.

## What Data is Collected

Each time your extension loads, a single JSON payload is sent:

```json
{
  "extension_name": "your_extension",
  "extension_version": "v1.0.0",
  "user_agent": "query-farm/20260201",
  "duckdb_platform": "linux_amd64",
  "duckdb_library_version": "v1.2.1",
  "duckdb_release_codename": "Kirimomi",
  "duckdb_source_id": "abc123"
}
```

| Field | Description |
|---|---|
| `extension_name` | The name of the extension being loaded |
| `extension_version` | The version string of the extension |
| `user_agent` | Fixed identifier for the telemetry client version |
| `duckdb_platform` | The DuckDB platform string (OS + architecture, e.g. `linux_amd64`, `osx_arm64`) |
| `duckdb_library_version` | The DuckDB library version (e.g. `v1.2.1`) |
| `duckdb_release_codename` | The DuckDB release codename |
| `duckdb_source_id` | The DuckDB source commit hash |

### What is NOT collected

- No personally identifiable information (PII)
- No queries or SQL statements
- No file paths or database contents
- No device IDs or hardware identifiers
- No IP-derived geolocation (IPs are not stored)
- No tracking cookies or persistent identifiers

## How to Opt Out

Set the `QUERY_FARM_TELEMETRY_OPT_OUT` environment variable to any value before loading the extension:

```bash
export QUERY_FARM_TELEMETRY_OPT_OUT=1
```

When this variable is set, `QueryFarmSendTelemetry()` returns immediately without sending any data.

## Integration Guide

### Step 1: Copy the source files

Copy the two files into your DuckDB extension's source tree:

```
your-extension/
  src/
    include/
      query_farm_telemetry.hpp   <-- copy here
    query_farm_telemetry.cpp     <-- copy here
```

### Step 2: Add to CMakeLists.txt

Add `query_farm_telemetry.cpp` to your extension's source list. For example, if your `CMakeLists.txt` uses a file list:

```cmake
set(EXTENSION_SOURCES
    src/your_extension.cpp
    src/query_farm_telemetry.cpp
)
```

### Step 3: Include the header

In your extension's main source file (where your `Load` function is defined):

```cpp
#include "query_farm_telemetry.hpp"
```

### Step 4: Call the telemetry function

Add a single call at the end of your extension's `Load` function:

```cpp
static void LoadInternal(ExtensionLoader &loader) {
    // ... your extension loading code ...

    QueryFarmSendTelemetry(loader, "your_extension_name", "v1.0.0");
}
```

The function takes three arguments:
- `loader` — the `ExtensionLoader` reference passed to your `Load` function
- `extension_name` — a string identifying your extension
- `extension_version` — the version string of your extension

That's it. The telemetry call is fire-and-forget and will not interfere with your extension's loading.

## How It Works

1. **Opt-out check** — If the `QUERY_FARM_TELEMETRY_OPT_OUT` environment variable is set, the function returns immediately.

2. **httpfs auto-load** — The client attempts to auto-load the `httpfs` extension (bundled with DuckDB) to gain HTTP capabilities. If `httpfs` cannot be loaded, the function silently returns without sending anything.

3. **JSON payload** — A JSON object is constructed using DuckDB's bundled yyjson library containing only the fields listed above.

4. **Async HTTP POST** — On native platforms, the HTTP request is dispatched via `std::async` so it runs in a background thread and does not block extension loading. On WASM (`__EMSCRIPTEN__`), the request is made synchronously since threading is not available.

5. **Silent error handling** — All exceptions from the HTTP request are caught and silently ignored. Telemetry must never cause an extension to fail.

6. **Symbol hiding** — The `INTERNAL_FUNC` macro applies `__attribute__((visibility("hidden")))` on GCC/Clang to prevent the telemetry function from appearing in the extension's public symbol table.

## Dependencies

**Zero external dependencies.** The client uses only:
- DuckDB core headers (available in every extension build)
- DuckDB's bundled yyjson library (for JSON serialization)
- The `httpfs` extension at runtime (auto-loaded, ships with DuckDB)
- C++ standard library (`<future>`, `<cstdlib>`)

## Privacy

Query.Farm telemetry is designed with privacy as a priority:

- **Minimal data** — Only platform and version information is collected. No user data, queries, or identifying information is sent.
- **Sent once per load** — The telemetry call fires once when the extension is loaded, not on every query or operation.
- **Opt-out available** — Set `QUERY_FARM_TELEMETRY_OPT_OUT` to any value to disable telemetry entirely.
- **Purpose** — The data is used solely for understanding which DuckDB platforms and versions are in use, to guide extension development and compatibility testing.

## License

MIT License. See [LICENSE](LICENSE) for details.
