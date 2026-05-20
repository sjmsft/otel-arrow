# df_engine_shim Logging & Telemetry Analysis

**Analysis Date**: May 4, 2026

## Overview

This document analyzes the logging capabilities and telemetry exposed by the df_engine_shim layer, which provides a C ABI interface for embedding otap-dataflow in C++ monitoring agents.

## 1. Log Level Hierarchy

The shim exposes 5 severity levels matching standard agent logging:

| Value | Level   | Usage                                    |
|-------|---------|------------------------------------------|
| 0     | Error   | Critical failures and errors             |
| 1     | Warn    | Warning conditions                       |
| 2     | Info    | Informational messages (default)         |
| 3     | Debug   | Detailed debugging information           |
| 4     | Trace   | Fine-grained trace-level diagnostics     |

## 2. Dual Logging Bridge Architecture

### A. Tracing Subscriber Bridge

**Source**: [crates/df_engine_shim_core/src/logging.rs](crates/df_engine_shim_core/src/logging.rs)

Routes all Rust `tracing` output from otap-dataflow internals (geneva-uploader, tonic, tokio, etc.) through the C++ host's logging infrastructure via `LogCallback`.

**Key Features**:
- **Structured Field Support**: Captures and formats multiple data types:
  - String values (`record_str`)
  - Integers (`record_u64`, `record_i64`)
  - Floats (`record_f64`)
  - Booleans (`record_bool`)
  - Debug representations (`record_debug`)

- **Event Formatting**: Constructs messages as:
  ```
  [target] message field1=value1 field2=value2
  ```

- **Filtering**: Respects `RUST_LOG` environment variable with default level of `info`

- **Installation**: Process-wide singleton subscriber installed before pipeline starts

- **Smart Content Detection**: Skips events with no useful content; avoids tracing's default "event src/file:line" names

### B. Callback Exporter (Internal Telemetry System)

**Source**: [crates/df_engine_shim_core/src/callback_exporter.rs](crates/df_engine_shim_core/src/callback_exporter.rs)

Routes otap-dataflow internal telemetry (ITS) through the host callback.

**Configuration**:
- **URN**: `urn:df_engine_shim:exporter:callback`
- **Purpose**: Sink for `urn:otel:receiver:internal_telemetry` in `engine.observability.pipeline`
- **Protocol**: Decodes OTLP ExportLogsServiceRequest protobuf

**Severity Mapping** (OTLP → DfLogLevel):
| OTLP Severity | DfLogLevel |
|---------------|------------|
| 0-4           | Trace      |
| 5-8           | Debug      |
| 9-12          | Info       |
| 13-16         | Warn       |
| 17+           | Error      |

**Message Format**: Includes instrumentation scope name:
```
[scope_name] body
```

## 3. Telemetry Signal Support

### Currently Implemented ✅

**Logs**: Full support via both tracing bridge and callback exporter
- Decodes OTLP log records
- Extracts severity, body, and scope information
- Routes through host's LogCallback
- Supports structured fields and attributes

### Recognized but Not Yet Implemented ⏳

**Metrics**: Code recognizes `ExportMetricsRequest` but includes TODO:
```rust
// TODO: Forward metrics via host callback once host-side handling is defined.
```

**Traces**: Code recognizes `ExportTracesRequest` but includes TODO:
```rust
// TODO: Forward traces via host callback once host-side handling is defined.
```

## 4. Callback Registration System

The shim provides two scopes of log callback registration:

### Global Callback
```c
void df_engine_register_global_log_callback(df_log_callback_t log_fn);
```

**Characteristics**:
- For messages not associated with specific engine instances
- Process-wide, captures third-party library logs
- Must be called before `df_engine_init` to capture global diagnostics
- Only one active at a time (subsequent calls replace previous)
- Warns on the previous callback before replacing

### Per-Instance Callback

Registered via `df_engine_init()` parameters:
```c
DfEngineHandle* df_engine_init(const char *yaml_config_utf8, 
                               df_log_callback_t log_fn);
```

**Characteristics**:
- Scoped to individual engine handles
- Keyed by monotonic instance ID
- Allows multi-instance hosts to separate logs
- Automatically registered and unregistered with handle lifecycle

## 5. C Callback Signature

```c
typedef void (*df_log_callback_t)(enum DfLogLevel level, const char *message);
```

**Safety Contract**:
- The `message` pointer is valid only for the duration of the callback call
- The host must not store the pointer
- Host should copy the message if persistence is needed

## 6. Health Monitoring API

```c
int32_t df_engine_health(DfEngineHandle *handle, const char **output);
```

**Current Functionality**:
- Returns JSON string with engine status
- Current schema: `{"status":"ok","pipelines":0}`
- Error codes: Returns `-1` for invalid handle/null pointers
- TODO: Query the controller's actual health endpoint

**Usage**:
```c
const char *health_json = NULL;
int rc = df_engine_health(handle, &health_json);
if (rc == 0) {
    // Process health_json
    df_engine_free_string(health_json);  // Must free!
}
```

## 7. Version Introspection

### Version String
```c
const char* df_engine_get_version(void);
```
Returns runtime library version string.

### Component Versions
```c
int32_t df_engine_get_component_versions(const char **output);
```
Returns JSON with shim/otap/otel-arrow version metadata. Must free result with `df_engine_free_string()`.

## 8. Key Design Features

### 1. Zero-Copy Integration
- LogCallback passes `const char*` valid only during callback duration
- No unnecessary string allocations or copies
- Host controls memory management for persisted messages

### 2. Instance Isolation
- Each `DfEngineHandle` gets unique monotonic instance ID
- Callbacks and channels scoped per instance
- Enables multi-instance hosts with separate log streams

### 3. Thread-Safety
- Uses `Mutex<LogCallbackRegistry>` for concurrent callback access
- Process-wide `LazyLock` for singleton initialization
- `Once` for tracing subscriber installation

### 4. Fail-Safe Operation
- Skips events with no useful content
- Logs errors through host callback before returning error codes
- Graceful degradation when callback registration fails

### 5. Process-Wide Coordination
- Single tracing subscriber for all instances
- Fallback to global callback for unscoped messages
- Crypto provider initialization shared across instances

## 9. Example Pipeline Configuration for ITS

To enable internal telemetry routing through the callback exporter:

```yaml
version: otel_dataflow/v1
engine:
  observability:
    pipeline:
      nodes:
        internal_receiver:
          type: "urn:otel:receiver:internal_telemetry"
          config: {}
        callback_exporter:
          type: "urn:df_engine_shim:exporter:callback"
          config: {}
      connections:
        - from: internal_receiver
          to: callback_exporter
groups:
  mygroup:
    pipelines:
      main:
        nodes:
          # ... application pipeline nodes ...
```

## 10. Integration with Host Agents

### For mdsd (Linux) / MonAgent (Windows)

**Host Responsibilities**:
1. Implement `df_log_callback_t` to bridge into host logging system
2. Register global callback before initialization (optional)
3. Pass per-instance callback to `df_engine_init`
4. Copy message strings if persistence beyond callback scope is needed
5. Free health/version JSON strings with `df_engine_free_string`

**Benefits**:
- Unified log stream: otap-dataflow diagnostics appear in host agent logs
- Consistent severity mapping across Rust and C++ components
- No separate log file management for embedded df_engine
- Instance-scoped routing for multi-pipeline deployments

## 11. Future Telemetry Expansion

### Planned Additions
- **Metrics forwarding**: Export internal pipeline metrics (throughput, latency, errors) via host callback
- **Trace forwarding**: Export distributed traces for pipeline operations
- **Enhanced health endpoint**: Query actual controller health with pipeline details

### Host Callback Extensions (Future)
```c
// Potential future callbacks
typedef void (*df_metrics_callback_t)(const char *metrics_json);
typedef void (*df_trace_callback_t)(const char *trace_json);
```

## Summary

The df_engine_shim layer provides comprehensive logging capabilities that enable C++ monitoring agents to seamlessly integrate otap-dataflow diagnostics:

- **Complete log signal support** via dual bridges (tracing + ITS callback exporter)
- **Structured logging** with field capture and formatting
- **Flexible scoping** (global + per-instance callbacks)
- **Thread-safe** concurrent operation
- **Zero-copy** efficient integration
- **Future-ready** architecture for metrics and traces

This design ensures that monitoring agents maintain full visibility into embedded dataflow pipeline operations while preserving clean architectural boundaries between Rust and C++ components.
