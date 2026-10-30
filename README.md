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

That's it. The telemetry call never blocks or fails your extension's loading. Closing the database (or exiting the process) waits for an in-flight ping, bounded by a 3 second request timeout; on a normal connection that is one HTTP round trip.

**Requires DuckDB v1.5.0 or newer** (the client relies on `ObjectCacheEntry::GetEstimatedCacheMemory`, added in v1.5.0).

### Optional: sending other events

An extension can send its own events (vgi sends one per `ATTACH`) through the same
transport, with the same opt-out, httpfs handling, and shutdown guarantees:

```cpp
auto doc = yyjson_mut_doc_new(nullptr);
auto root = yyjson_mut_obj(doc);
yyjson_mut_doc_set_root(doc, root);
QueryFarmAddBaseEnvelope(doc, root, "your_extension", "v1.0.0"); // the 7 base fields
yyjson_mut_obj_add_str(doc, root, "event", "attach");
// ... serialize `doc` to `json` ...
QueryFarmSendEventJson(db, "your_extension", "https://your-endpoint/", json, QueryFarmHttpfsAutoload::DISALLOW);
```

Pass `QueryFarmHttpfsAutoload::DISALLOW` on any per-query path: auto-loading httpfs is synchronous and
can download an extension, so the event is dropped instead if httpfs is not already loaded. Each
event runs on its own worker; finished workers are reaped as new ones start, and the rest are joined
when the database closes or the process exits.

## How It Works

1. **Opt-out check** — If the `QUERY_FARM_TELEMETRY_OPT_OUT` environment variable is set, the function returns immediately.

2. **httpfs auto-load** — The client attempts to auto-load the `httpfs` extension (bundled with DuckDB) to gain HTTP capabilities. If `httpfs` cannot be loaded, the function silently returns without sending anything.

3. **JSON payload** — A JSON object is constructed using DuckDB's bundled yyjson library containing only the fields listed above.

4. **Parameters and client on the loading thread** — HTTP settings, proxy, and secrets are resolved and the HTTP client is constructed synchronously during load (no network I/O). Constructing the client is what initialises curl/OpenSSL inside `httpfs`, which matters for step 6.

5. **Background HTTP POST, owned by the database** — On native platforms the request runs on a `std::thread` that is owned by an entry in the database's `ObjectCache`, so `~DatabaseInstance` joins it before tearing down the HTTP machinery. The thread never holds a reference to the database. On WASM (`__EMSCRIPTEN__`), the request is made synchronously since threading is not available.

6. **Join at process exit** — A process can call `exit()` with the database still alive (the DuckDB shell does so on any non-zero exit). A process-wide `atexit` handler joins any in-flight worker. It is registered only after the client has been constructed, so it lands *after* OpenSSL's own `atexit(OPENSSL_cleanup)` and therefore runs *before* it; otherwise the worker could still be inside a TLS request while OpenSSL is being torn down, which segfaults on shutdown.

7. **Silent error handling** — Everything after the `httpfs` check runs inside a `try`/`catch`: a payload or HTTP-parameter error (for example an `http_proxy` value DuckDB cannot parse) or any exception from the request itself is swallowed. Telemetry must never cause an extension to fail to load.

8. **Symbol hiding** — The `INTERNAL_FUNC` macro applies `__attribute__((visibility("hidden")))` on GCC/Clang to prevent the telemetry function from appearing in the extension's public symbol table.

## Dependencies

**Zero external dependencies.** The client uses only:
- DuckDB core headers (available in every extension build)
- DuckDB's bundled yyjson library (for JSON serialization)
- The `httpfs` extension at runtime (auto-loaded, ships with DuckDB)
- DuckDB v1.5.0 or newer
- C++ standard library (`<thread>`, `<mutex>`, `<cstdlib>`)

## Privacy

Query.Farm telemetry is designed with privacy as a priority:

- **Minimal data** — Only platform and version information is collected. No user data, queries, or identifying information is sent.
- **Sent once per load** — The telemetry call fires once when the extension is loaded, not on every query or operation.
- **Opt-out available** — Set `QUERY_FARM_TELEMETRY_OPT_OUT` to any value to disable telemetry entirely.
- **Purpose** — The data is used solely for understanding which DuckDB platforms and versions are in use, to guide extension development and compatibility testing.

## License

MIT License. See [LICENSE](LICENSE) for details.
