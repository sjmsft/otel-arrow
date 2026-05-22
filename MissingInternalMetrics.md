# [Controller] Expose in-process metrics handle from observer startup callback

### Pre-filing checklist

- [x] I searched existing issues and didn't find a duplicate

### Component(s)

Rust OTAP dataflow (`rust/otap-dataflow/`)

### Objective

Expose an in-process handle for reading internal engine metrics snapshots from
the observer startup callback, so that hosting processes can access metrics
without going through the admin HTTP API.

### Rationale

Commit `32bc2ca5` (#2505) added `run_forever_with_observer` and
`run_till_shutdown_with_observer` to `Controller`, which invoke a
`FnOnce(ObservedStateHandle)` callback to give embedders in-process access to
pipeline liveness and health state.

However, the callback only provides `ObservedStateHandle` — it does not expose
the `TelemetryRegistryHandle` that the controller constructs internally during
startup. `TelemetryRegistryHandle` is the read-side aggregation store that the
admin HTTP server already uses to serve `/api/v1/telemetry/metrics` via its
`visit_current_metrics()` and `visit_metrics_and_reset()` visitor methods. Without
access to this handle, hosting processes that embed the engine as a library
cannot read internal metrics snapshots (e.g. per-node throughput, queue depths,
processing latencies) without enabling the admin HTTP server and querying it over
the network.

This is the metrics-access counterpart to the liveness/health access that
`ObservedStateHandle` already provides.

### Scope

- Extend the observer callback (or add a new callback variant) to also provide a
  metrics handle alongside `ObservedStateHandle`. This could be a combined
  struct, a second callback parameter, or a new `run_*_with_metrics_observer`
  method family.
- The exposed handle should allow callers to read the current metrics snapshot
  without requiring the admin HTTP server to be enabled.

### Acceptance Criteria

- Embedding binaries can obtain an in-process metrics handle from the controller
  startup callback.
- Metrics snapshots can be read without enabling or depending on `http_admin`.
- Existing `run_forever` and `run_till_shutdown` behavior is unchanged.

### Why the HTTP admin API is insufficient

Requiring embedding processes to query the admin HTTP API for metrics that
already live in the same process introduces performance overhead.

This is the same reasoning behind the existing `ObservedStateHandle` for
liveness/health: it avoids the HTTP path by returning snapshots directly at 
near-zero cost. The metrics handle should follow the same pattern.

### Dependencies or Blockers

Depends on the observer API introduced in commit `32bc2ca5` (#2505).
