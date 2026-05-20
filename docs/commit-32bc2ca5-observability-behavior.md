# Commit 32bc2ca5 — Observability Behavior Analysis

**Commit:** `32bc2ca50c1138e94719947542027fff77c39c1b`
**Title:** feat(otap-dataflow): expose startup helpers as a public library module (#2505)
**Author:** Lalit Kumar Bhasin
**Date:** 2026-04-06
**Fixes:** #2462, #2463

---

## Summary

This commit restructures the OTAP dataflow engine to support library-mode
embedding. It moves startup functions into a public `otap_df_controller::startup`
module and adds observer-pattern APIs to the `Controller` for in-process health
monitoring. The commit does **not** expose the engine's internal telemetry
subsystem (MeterProvider, TracerProvider, metrics pipeline) to the host
application.

---

## What Is Exposed to the Host

### 1. `ObservedStateHandle` (Pipeline Health State)

The commit adds two new public methods on `Controller`:

- `run_forever_with_observer(engine_config, observer_callback)`
- `run_till_shutdown_with_observer(engine_config, observer_callback)`

Both accept a `FnOnce(ObservedStateHandle)` callback that fires once, before
the engine blocks, giving the host a handle to the pipeline state store.

**`ObservedStateHandle` provides:**

| Method | Returns | Description |
|---|---|---|
| `snapshot()` | `HashMap<PipelineKey, PipelineStatus>` | Cloned snapshot of all pipeline statuses |
| `pipeline_status(key)` | `Option<PipelineStatus>` | Status of a single pipeline |
| `liveness(key)` | `bool` | Whether a pipeline is considered live |
| `readiness(key)` | `bool` | Whether a pipeline is considered ready |

This is **operational health state** — liveness, readiness, and pipeline status —
not telemetry data. It is backed by an `Arc<Mutex<HashMap>>` shared with the
internal `ObservedStateStore`, offering zero-overhead, in-process access
without going through the admin HTTP server.

### 2. `startup` Module (Reusable Bootstrap Helpers)

The following functions moved from `src/main.rs` to
`otap_df_controller::startup`:

| Function | Purpose |
|---|---|
| `core_allocation_override(num_cores, core_id_range)` | Resolves CLI core-allocation flags into a `CoreAllocation` |
| `http_admin_bind_override(bind_address)` | Converts an optional bind-address string into `HttpAdminSettings` |
| `apply_cli_overrides(cfg, num_cores, core_id_range, http_admin_bind)` | Merges CLI flags into a parsed `OtelDataflowSpec` |
| `validate_pipeline_components(group_id, pipeline_id, pipeline_cfg, factory)` | Validates that all node URNs in a pipeline map to registered components |
| `validate_engine_components(engine_cfg, factory)` | Top-level validation across all pipeline groups and observability pipeline |
| `system_info(factory, memory_allocator)` | Returns a diagnostic string with CPU, memory, build mode, and registered component URNs |

These are **static validation and diagnostics** helpers. They do not produce
or expose runtime telemetry.

---

## What Is NOT Exposed

The following remain internal to the controller and are **not** accessible to
the host application:

| Component | Internal Location | Description |
|---|---|---|
| `TelemetryRegistryHandle` | Created in `run_with_mode()` | Central registry for all internal metrics |
| `InternalTelemetrySystem` | Created in `run_with_mode()` | Manages the full internal telemetry pipeline (metrics dispatching, tracing, log providers) |
| `MetricsReporter` / `MetricsDispatcher` | Obtained from `InternalTelemetrySystem` | Internal metrics collection and reporting |
| `ObservedEventReporter` | Created from `ObservedStateStore` | Event reporting channel for pipeline state transitions |
| Admin tracing setup | Via `telemetry_system.admin_tracing_setup()` | Tracing configuration for the admin HTTP server |
| Internal tracing setup | Via `telemetry_system.internal_tracing_setup()` | Tracing configuration for engine internals |
| OTel SDK providers | Encapsulated within `InternalTelemetrySystem` | MeterProvider, TracerProvider, LoggerProvider — none are returned or exposed via public API |

The `InternalTelemetrySystem`, along with its SDK providers and metrics
pipeline, is created after the observer callback fires and remains fully
encapsulated within the controller's runtime loop.

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│  Host Application                                                │
│                                                                  │
│   Controller::run_forever_with_observer(config, |handle| {       │
│       // handle: ObservedStateHandle                             │
│       //   .snapshot()       → pipeline statuses                 │
│       //   .liveness(key)    → bool                              │
│       //   .readiness(key)   → bool                              │
│   })                                                             │
│                                                                  │
│   startup::validate_engine_components(...)                       │
│   startup::apply_cli_overrides(...)                              │
│   startup::system_info(...)                                      │
└──────────────────────────────┬───────────────────────────────────┘
                               │ ObservedStateHandle
                               │ (Arc<Mutex<HashMap>>)
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│  Controller Internals (not exposed)                              │
│                                                                  │
│   ObservedStateStore ◄─── pipeline status updates                │
│       │                                                          │
│       ├─► ObservedStateHandle (shared with host)                 │
│       └─► ObservedEventReporter (internal only)                  │
│                                                                  │
│   InternalTelemetrySystem (internal only)                        │
│       ├─► TelemetryRegistryHandle                                │
│       ├─► MetricsDispatcher / MetricsReporter                    │
│       ├─► Admin tracing setup                                    │
│       ├─► Internal tracing setup                                 │
│       └─► OTel SDK providers (Meter/Tracer/LoggerProvider)       │
└──────────────────────────────────────────────────────────────────┘
```

---

## Example Usage (from `examples/custom_collector.rs`)

```rust
let controller = Controller::new(&OTAP_PIPELINE_FACTORY);
controller.run_forever_with_observer(engine_cfg, |handle| {
    eprintln!("[observer] ObservedStateHandle obtained");
    std::thread::spawn(move || {
        loop {
            std::thread::sleep(std::time::Duration::from_secs(5));
            let snapshot = handle.snapshot();
            for (key, status) in &snapshot {
                eprintln!(
                    "[observer] pipeline {}:{} -> {:?}",
                    key.pipeline_group_id().as_ref(),
                    key.pipeline_id().as_ref(),
                    status
                );
            }
        }
    });
});
```

The observer callback receives an `ObservedStateHandle` and spawns a
background thread that periodically polls pipeline health. This is the
full extent of runtime observability available to the host — there is no
access to internal metrics, traces, or log telemetry.

---

## Gap: Internal Telemetry Not Exposed

For a host application to access the engine's internal telemetry (e.g., to
feed engine metrics into the host's own MeterProvider, or to correlate engine
traces with application traces), additional API surface would be needed.
Potential approaches include:

1. **Exposing the `TelemetryRegistryHandle`** via the observer callback or a
   dedicated method, allowing the host to read registered metric instruments.
2. **Accepting a host-provided MeterProvider/TracerProvider** at `Controller`
   construction time, so the engine records its internal telemetry into the
   host's SDK instance.
3. **Returning an `InternalTelemetryHandle`** that exposes read-only access to
   the metrics reporter or a subset of the telemetry registry.

None of these exist in the codebase as of this commit.
