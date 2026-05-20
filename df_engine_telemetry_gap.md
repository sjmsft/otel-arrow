# df_engine (otap-dataflow) — Telemetry Gaps for MA Integration

**Date:** May 8, 2026

**Context:** The `df_engine_shim` layer provides a C ABI for embedding
`otap-dataflow` in a C++ monitoring agent (MA). It depends on commit
`32bc2ca5` (library-mode embedding) and bridges engine internals to MA's
four telemetry sinks: `MaEventTable`, `MaQosTable`, `AgentMetrics`, and
`MaTelemetryEvents`. This document identifies the work remaining **in the
otap-dataflow project** so that the shim can satisfy all MA telemetry
requirements.

**Reference documents:**

- `MaTelemetryRequirements.md` — MA sink schemas and `ITelemetryListener` design.
- `df_engine_shim_telemetry_current.md` — current shim capabilities.
- `df_engine_shim_telemetry_gap.md` — shim-side gap analysis.
- `docs/commit-32bc2ca5-observability-behavior.md` — what library mode exposes today.

---

## Current State Summary

### What the engine already provides

| Capability | Mechanism | Notes |
|---|---|---|
| **Internal logs** | `InternalTelemetrySystem` → ITS channel → `internal_telemetry_receiver` → observability pipeline → callback exporter | Shim bridges these to `MaEventTable` via `df_log_callback_t`. Working. |
| **Tracing subscriber bridge** | `tracing` crate subscriber installed by shim | Routes Rust `tracing` events to host `LogCallback`. Working. |
| **Pipeline health state** | `ObservedStateHandle` via `run_*_with_observer()` | Liveness, readiness, pipeline status. Usable for health endpoints. |
| **Rich internal metrics registry** | `TelemetryRegistryHandle` with `visit_metrics_and_reset()` | 40+ `#[metric_set]` definitions across engine, pipeline, channel, and node levels. |
| **Metrics → OTel SDK path** | `MetricsDispatcher` flushes registry to `SdkMeterProvider` via OTLP | Works for standalone OTLP export to backends. **Not usable by the shim** because the `SdkMeterProvider` and dispatcher are internal to the controller. |
| **ITS logs-only channel** | `InternalTelemetrySettings` carries only `logs_receiver: Receiver<ObservedEvent>` | The observability pipeline receives **only logs**, not metrics or traces. |

### What is missing

The shim cannot currently satisfy `MaQosTable`, `AgentMetrics`, or
`MaTelemetryEvents` requirements because the engine does not expose the
data in a form the shim can consume.

---

## Gap 1: Internal Metrics Not Routed to the Observability Pipeline

**Problem:** The `internal_telemetry_receiver` only processes
`ObservedEvent::Log` events. It explicitly ignores `CollectTelemetry`
control messages ("No metrics to report for now"). The engine's metrics
flow through `MetricsDispatcher → SdkMeterProvider → OTLP exporter`
which is a standalone export path — it does not feed into the
observability pipeline where the shim's `callback_exporter` sits.

**Impact:** The shim's `callback_exporter` recognizes
`ExportMetricsRequest` but never receives one. All metrics that could map
to `AgentMetrics` (MDM) or `MaTelemetryEvents` counters are invisible to
the host.

**Required work:**

1. **Extend `InternalTelemetrySettings`** to include a metrics channel
   (or reuse the existing `MetricsReporter` channel) so that aggregated
   metric snapshots can be forwarded to the observability pipeline.
2. **Extend `internal_telemetry_receiver`** to handle
   `CollectTelemetry` by encoding accumulated metric snapshots into
   OTLP `ExportMetricsServiceRequest` and publishing them as `OtapPdata`
   alongside the log events.
3. **Alternatively,** provide a separate `MetricsReceiver` node in the
   observability pipeline, or expose a `visit_metrics` callback to the
   shim via the controller API (see Gap 4).

**Effort:** Medium. The metric data is already collected and aggregated
in `TelemetryRegistryHandle`; the gap is plumbing it into the
observability pipeline.

---

## Gap 2: No QoS-Level Operation Telemetry

**Problem:** MA's `MaQosTable` expects per-operation records with
`(operation, object, success, result, retries, durationMs,
dataSizeBytes, itemsRead, itemsWritten, startTime, endTime)`. The
engine's existing metrics are **aggregated counters and gauges** (e.g.
`log_batches_uploaded: Counter<u64>`, `log_upload_success_duration:
Mmsc`). There is no mechanism to emit a **per-operation event** with the
full QoS field set.

**Impact:** The shim has no `df_qos_callback_t` and no source data to
populate one. `MaQosTable` remains empty.

**Required work:**

1. **Define a QoS event type** in the telemetry crate — a struct
   capturing `operation`, `object`, `success`, `result_code`, `retries`,
   `duration_ms`, `data_size_bytes`, `items_read`, `items_written`,
   `start_time`, `end_time`.
2. **Emit QoS events from exporters.** The `geneva_exporter` already
   tracks per-upload duration and batch/byte counts internally; it needs
   to additionally produce a QoS event per upload attempt. Similarly for
   `otlp_grpc_exporter` and `otap_exporter`.
3. **Emit QoS events from receivers.** The OTLP receiver tracks
   `requests_started`, `requests_completed`, and `request_bytes` as
   counters; per-request QoS records are needed.
4. **Route QoS events** through the observability pipeline (or a
   dedicated channel) to the shim's callback exporter.
5. **Shim-side:** Define `df_qos_callback_t` and map QoS events to
   `LogQosEvent` calls. (Out of scope for this project, but the engine
   must provide the data.)

**Effort:** Medium-High. This is new event-level instrumentation, not
just plumbing existing data.

---

## Gap 3: No Structured Telemetry Events for `MaTelemetryEvents`

**Problem:** MA's `MaTelemetryEvents` table expects structured
per-operation records like `RowsSent`, `RowsReceived`, `RowsDropped`,
`Errors`, `ServiceRequest`, `ComponentStart`, and `Heartbeat`, each with
operation-specific dimension sets. The engine has no concept of these
structured event types.

**Impact:** The `MaTelemetryEvents` table remains empty. The host cannot
report data-flow telemetry to Geneva common agent telemetry.

**Required work:**

### 3a. `RowsSent` / `RowsReceived` / `RowsDropped`

The engine already tracks:
- `flow.signals.incoming` / `flow.signals.outgoing` (flow_metrics)
- `exporter.pdata.{logs,metrics,traces}_{consumed,exported,failed}`
- `geneva_exporter.log_records_uploaded`, `log_bytes_uploaded`, etc.
- `batch_processor.consumed_batches_*`, `produced_batches_*`

**What's missing:** These are aggregated counters. To produce
`RowsSent`/`RowsReceived` events, the engine needs to emit
**per-batch/per-upload structured events** with:
- `DataStartTimeUtc` / `DataEndTimeUtc`
- `OperationDurationMs`
- `DataSizeInBytes` / `BlobSizeInBytes`
- `RowsRead` (item count)
- `Success` / `HttpStatus`
- `DataAgeMs`

The `geneva_exporter` is closest to having this data (it tracks duration,
bytes, and record counts per upload). It needs to emit a structured event
in addition to updating counters.

### 3b. `Errors`

The engine logs errors through `tracing` and `otel_error!` macros. These
reach the host as log messages but lack the `ErrorType` / `ErrorSubtype` /
`HealthStatus` classification that `MaTelemetryEvents::Errors` requires.

**Required:** An error classification taxonomy for engine errors and
structured error events emitted alongside log messages.

### 3c. `ServiceRequest`

The OTLP receiver, OTAP receiver, and gRPC exporters make service calls
but do not emit structured `ServiceRequest` events with
`ResponseSizeInBytes`, `Success`, `HttpStatus`, `ApiName`, `Url`,
`AuthType`.

**Required:** Instrumentation in gRPC client/server code to emit
service-request events.

### 3d. `ComponentStart` / `Heartbeat`

- **ComponentStart:** The controller logs startup events but does not
  emit a structured `ComponentStart` with `Success`/`Reason`.
- **Heartbeat:** No periodic heartbeat event is emitted. The health
  endpoint (`ObservedStateHandle`) provides this data but not as a
  structured event.

**Effort:** High. This is the largest gap — it requires new
instrumentation throughout the pipeline and a new structured event
framework.

---

## Gap 4: Controller Does Not Expose Telemetry to the Host

**Problem:** Commit `32bc2ca5` exposes `ObservedStateHandle` (pipeline
health) but not any telemetry-related types. The following remain
internal:

| Type | What it provides |
|---|---|
| `TelemetryRegistryHandle` | Access to all registered metrics via `visit_metrics_and_reset()` |
| `InternalTelemetrySystem` | The full ITS pipeline, SDK providers, dispatchers |
| `MetricsReporter` | Channel for reporting metric snapshots |
| `MetricsDispatcher` | Periodic flush of registry to OTel SDK |
| `SdkMeterProvider` | OTel SDK meter provider |

**Impact:** Even if the engine emits all the right data internally, the
shim cannot access it except through the observability pipeline (which
currently only carries logs).

**Required work (options, not mutually exclusive):**

### Option A: Extend the observer callback

Add telemetry handles to the observer callback:

```rust
pub fn run_forever_with_observer<F>(
    &self,
    engine_config: OtelDataflowSpec,
    observer: F,
) -> Result<(), Error>
where
    F: FnOnce(ObservedStateHandle, TelemetryRegistryHandle),
```

This would let the shim call `visit_metrics_and_reset()` on a timer to
pull metric snapshots. **Simplest option** but requires the shim to do
its own polling and OTLP encoding.

### Option B: Accept a host-provided MeterProvider

Allow the `Controller` to accept an externally-provided
`SdkMeterProvider` so that all engine metrics are recorded into the
host's SDK instance:

```rust
pub fn with_meter_provider(mut self, provider: SdkMeterProvider) -> Self
```

The shim could create a `SdkMeterProvider` with a custom exporter that
routes to the host callback.

### Option C: Observability pipeline for all signals

Ensure the observability pipeline carries metrics and traces (not just
logs) and let the `callback_exporter` handle all three signal types. This
is the most architecturally aligned approach (uses existing pipeline
machinery) but requires Gap 1 to be resolved first.

**Recommendation:** Option C for metrics (architecturally clean),
supplemented by Option A for the structured event types (QoS,
`MaTelemetryEvents`) that don't fit the OTLP metrics model.

**Effort:** Medium.

---

## Gap 5: No Trace Context Propagation to Host

**Problem:** `MaEventTable` and `MaTelemetryEvents` both use
`ActivityId` / `traceparent` for cross-component correlation. The engine
has internal tracing (`TracingSetup`), but no trace context is propagated
through the shim's callback interface.

**Required work:**

1. The shim's `df_log_callback_t(level, message)` signature lacks a
   `traceparent` field. The engine could include `traceparent` in the
   message or in a structured field.
2. For QoS and structured telemetry events, include `traceparent` as a
   field.
3. The `internal_telemetry_receiver` could propagate the span context
   from log events into the OTLP log records (partially done — the ITS
   tracing subscriber captures span context for `tracing` events).

**Effort:** Low-Medium.

---

## Gap 6: ITS `callback_exporter` Metrics/Traces Paths Are Stubbed

**Problem:** The shim's `callback_exporter.rs` recognizes
`ExportMetricsRequest` and `ExportTracesRequest` but contains TODO
comments instead of forwarding logic. Even if the engine starts sending
metrics through the observability pipeline (Gap 1), the shim-side
exporter won't process them.

**Required work (shim-side, listed here for completeness):**

1. Implement `ExportMetricsRequest` decoding → extract metric data
   points → invoke `df_metric_callback_t`.
2. Implement `ExportTracesRequest` decoding → extract spans → invoke
   `df_trace_callback_t` (lower priority).
3. Define the ABI for passing metric dimensions across FFI (parallel
   `const char*` arrays or a serialized format).

**Effort:** Medium. (Shim-side work, but blocked on Gap 1 in this
project.)

---

## Gap 7: Log Callback Enrichment

**Problem:** The current log callback signature is
`df_log_callback_t(level, message)`. `MaEventTable` also accepts
`ErrorCode`, `MDRESULT`, and source location (`File`/`Function`/`Line`).

The engine's `tracing` subscriber captures `target` but not Rust source
file/line across FFI. The `otel_error!` macro does not carry a numeric
error code.

**Required work:**

1. **Engine-side:** Optionally attach error codes to `ObservedEvent::Log`
   events (e.g. an `error_code: Option<u32>` field).
2. **Engine-side:** Include source location (`file!()`, `line!()`) in
   log events when available.
3. **Shim-side:** Extend `df_log_callback_t` to
   `(level, message, error_code, file, line)` or use a struct-based ABI.

**Effort:** Low.

---

## Priority-Ordered Work Plan

| Priority | Gap | Deliverable | Blocks |
|---|---|---|---|
| **P0** | Gap 1 | Route internal metrics through the observability pipeline | All metrics-based sinks |
| **P0** | Gap 4 (Option C) | Ensure callback exporter receives metrics | `AgentMetrics`, `MaTelemetryEvents` counters |
| **P1** | Gap 2 | QoS event type + exporter/receiver instrumentation | `MaQosTable` |
| **P1** | Gap 3a | Structured `RowsSent`/`RowsReceived`/`RowsDropped` events | `MaTelemetryEvents` data-flow operations |
| **P1** | Gap 7 | Log event enrichment (error code, source location) | Full `MaEventTable` coverage |
| **P2** | Gap 3b | Error classification taxonomy | `MaTelemetryEvents::Errors` |
| **P2** | Gap 3c | ServiceRequest events from gRPC nodes | `MaTelemetryEvents::ServiceRequest` |
| **P2** | Gap 5 | Trace context propagation in callbacks | Cross-sink correlation |
| **P3** | Gap 3d | ComponentStart / Heartbeat events | `MaTelemetryEvents` lifecycle |
| **P3** | Gap 6 | Shim-side metrics/traces decoding (shim repo) | End-to-end metrics delivery |

---

## Metrics Already Available in the Engine (Reference)

The engine already collects these metrics internally. Once Gap 1 and
Gap 4 are resolved, many can be mapped to MA sinks:

### Engine-Level (`engine.metrics`)
- `memory_rss`, `cpu_utilization`, `memory_pressure_state`
- `process_memory_usage_bytes`, `process_memory_soft_limit_bytes`,
  `process_memory_hard_limit_bytes`

### Pipeline-Level (`pipeline.metrics`)
- `uptime`, `memory_usage`, `memory_allocated`, `memory_freed`
- `cpu_time`, `cpu_utilization`
- `context_switches_voluntary`, `context_switches_involuntary`
- `page_faults_minor`, `page_faults_major`

### Channel-Level (`channel.sender` / `channel.receiver`)
- `send.count`, `send.error_full`, `send.error_closed`
- `recv.count`, `recv.error_empty`, `recv.error_closed`

### Node-Level (`node.consumer` / `node.producer`)
- `consumed.duration`, `consumed.success`, `consumed.failure`, `consumed.refused`
- `produced.duration`, `produced.success`, `produced.failure`, `produced.refused`

### Flow Metrics (`flow`)
- `signals.incoming`, `signals.outgoing`, `compute.duration`

### Exporter Metrics (`exporter.pdata`)
- Per-signal `consumed`, `exported`, `failed` counters

### Geneva Exporter (`otap.exporter.geneva`)
- Per-signal batch/record/byte upload counters
- Per-upload success/failure duration (Mmsc)
- Encode duration

### Receiver Metrics (`otlp.receiver`)
- `requests_started`, `requests_completed`, `rejected_requests`
- `refused_memory_pressure`, `transport_errors`, `request_bytes`

### Processor Metrics
- Batch: `consumed_batches_*`, `produced_batches_*`, flush metrics
- Retry: Per-signal `consumed_items_*_{success,failure,refused}`,
  `produced_items_*_{success,refused}`, `retry_attempts_*`
- Compute duration: `compute.success.duration`, `compute.failed.duration`

### Potential MA Sink Mappings

| Engine Metric | → MA Sink | → MA Field / Metric |
|---|---|---|
| `engine.metrics.cpu_utilization` | `MaTelemetryEvents` | `CpuUsage.OperationValue` |
| `engine.metrics.memory_rss` | `MaTelemetryEvents` | `MemoryUsage.OperationValue` |
| `exporter.pdata.logs_exported` | `AgentMetrics` | `EventsSent` |
| `exporter.pdata.logs_consumed` | `AgentMetrics` | `EventsLogged` |
| `exporter.pdata.logs_failed` | `AgentMetrics` | `EventsDropped` |
| `geneva.log_batches_uploaded` | `AgentMetrics` | `SucceededUploadTasks` |
| `geneva.log_batches_failed` | `AgentMetrics` | `FailedUploadTasks` |
| `otlp.receiver.request_bytes` | `MaQosTable` | `dataSizeInBytes` |
| `geneva.log_upload_success_duration` | `MaQosTable` | `durationInMilliseconds` |
| Per-upload event (Gap 2) | `MaTelemetryEvents` | `RowsSent.*` |
