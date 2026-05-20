# df_engine_shim vs. TelemetryListener Requirements — Gap Analysis

**Analysis Date:** May 8, 2026

**Sources:**

- [df_engine_shim_telemetry_current.md](df_engine_shim_telemetry_current.md) — current behavior of the df_engine shim layer.
- [MaTelemetryRequirements.md](MaTelemetryRequirements.md) — full requirements for the future `TelemetryListener` layer.

---

## Already Implemented in the Shim Layer

### 1. Diagnostic Logging (to `MaEventTable`)

The shim has solid coverage of the **logging** capability:

| Requirement | Shim Status |
| --- | --- |
| Severity levels (Error/Warn/Info/Debug/Trace) | Implemented. `DfLogLevel` enum maps cleanly to `MA_LEVEL_*` |
| Message text delivery | Implemented via `df_log_callback_t(level, message)` |
| Per-instance scoping (multiple streams) | Implemented. Per-handle callbacks via `df_engine_init` |
| Process-wide diagnostics | Implemented. `df_engine_register_global_log_callback` |
| Capture of Rust internals (tonic/tokio/geneva-uploader) | Implemented. Tracing subscriber bridge |
| Structured field formatting | Implemented. `record_str/u64/i64/f64/bool/debug` |
| Routing of OTLP-encoded internal log events | Implemented. Callback exporter (ITS) |
| OTLP severity to host severity mapping | Implemented |
| Thread safety | Implemented. `Mutex<LogCallbackRegistry>` + `LazyLock` |

This roughly satisfies the **`Log(level, message, ...)`** entry on `ITelemetryListener` — the host-side `TelemetryListener` can wrap the existing `df_log_callback_t` to call `IEventStream1::ExternalMessage`.

### 2. Version / Health Introspection (auxiliary, not in TelemetryListener spec)

- `df_engine_get_version`, `df_engine_get_component_versions`, `df_engine_health` — useful for agent diagnostics but not part of the telemetry sink contract.

---

## Missing / Not Yet Implemented

The shim today is essentially a **logs-only bridge**. The remaining three sinks defined in `MaTelemetryRequirements.md` have no shim surface.

### 1. QoS / Operation Telemetry (to `MaQosTable`)

- **Status:** Not implemented. The callback exporter contains a `TODO` for metrics and traces; QoS-style records have no dedicated path at all.
- **Missing:**
  - No `df_qos_callback_t` (or equivalent).
  - No way for the dataflow pipeline to emit `(operation, object, success, result, retries, dataDelayMs, durationMs, dataSizeBytes, itemsRead, itemsWritten, startTime, endTime)`.
  - No serialization contract (struct vs. JSON vs. protobuf) for crossing the FFI boundary.

### 2. MDM Aggregated Metrics (to `AgentMetrics`)

- **Status:** Not implemented. `ExportMetricsRequest` is recognized but discarded with a TODO.
- **Missing:**
  - No `df_metric_callback_t(name, value, event_name, dimensions...)`.
  - No mapping from OTLP metric data points (gauge/sum/histogram) to MDM `LogNumericMetric` form.
  - No mechanism to pass `AdditionalDimensions` (key/value pairs) across FFI safely.
  - No conventions for using canonical `MDM_METRIC_NAME_*` strings inside the pipeline.

### 3. Structured Operation Telemetry (to `MaTelemetryEvents`)

- **Status:** Not implemented. This is the largest gap.
- **Missing:**
  - No FFI surface for the operation-specific structs (`RowsSentInfo`, `RowsReceivedInfo`, `RowsDropped`, `Errors`, `ServiceRequest`, `ComponentStart`, `Heartbeat`).
  - No `df_telemetry_*` callback family.
  - No definition of how `Operation` + `OperationValue` + per-op dimension sets are emitted from the pipeline.
  - Agent-supplied common dimensions (Namespace, Region, AssetIdentity, etc.) are correctly *out of scope* for the shim — but the shim still needs to deliver the LIB-only fields.

### 4. Trace / Activity Correlation

- **Status:** Not implemented.
- **Missing:**
  - No propagation of `traceparent` / `ActivityId` from Rust pipeline operations to the host callback.
  - `MaEventTable` and `MaTelemetryEvents` both rely on thread-local activity IDs (`SetThreadActivityId` / `GetThreadTraceContexId`); there is no shim-side hook to plumb a span/trace id into callbacks.

### 5. Traces Signal (OTLP)

- **Status:** Recognized only.
- `ExportTracesRequest` decoding has a TODO; no host callback exists.

### 6. Source-Location Context for Logs

- **Status:** Partial.
- The tracing bridge captures `target` but does not forward `file` / `line` / `function` across FFI. `MaEventTable` requires these (`__WFILE__`, `__WFUNCTION__`, `__LINE__`); today the listener will have to substitute its own call-site, losing Rust-side origin.

### 7. Error-Code Channel

- **Status:** Not implemented.
- The log callback signature is `(level, message)` only. There is no field for `MDRESULT` or a numeric `ErrorCode`, both of which `MaEventTable` accepts. Adding these would let Rust errors be classified without parsing the message string.

### 8. Initialization / Lifecycle Hooks for Telemetry Sinks

- No shim API to register the four distinct callback families together (logs / QoS / MDM / structured telemetry).
- No equivalent of `ComponentStart` emission at `df_engine_init` / shutdown notification on handle drop.

---

## Suggested Next Increments (in priority order)

1. **Extend the log callback** to carry `error_code` (`u32`), `md_result` (`i32`), and optional `file`/`line`/`function` so the existing path fully covers `MaEventTable`.
2. **Add a QoS callback family** — a single `df_qos_callback_t` taking a POD struct mirroring `LogQosEvent` parameters. Lowest-complexity gap to close.
3. **Wire the OTLP metrics path** in `callback_exporter.rs` to a new `df_metric_callback_t` and define the dimension-passing ABI (e.g. parallel `const char* const*` arrays + length).
4. **Define structured telemetry callbacks** for the operation-specific cases (`RowsSent`, `RowsReceived`, `RowsDropped`, `Errors`, `ServiceRequest`). These map 1:1 to `ITelemetryListener::RecordRows*` / `RecordError` / `RecordServiceRequest`.
5. **Plumb trace context** through callbacks (add `traceparent` field) so all four sinks correlate.
6. **Traces forwarding** — last priority, since `MaTelemetryRequirements.md` does not require a trace sink, only `traceparent` strings.

---

## Summary

**Logging is largely done; QoS, MDM metrics, and `MaTelemetryEvents`-shaped structured telemetry are entirely missing**, and the existing log callback needs minor enrichment (error codes, source location, trace context) to fully satisfy `MaEventTable`.
