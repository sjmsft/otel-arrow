# Commit `32bc2ca5` — `feat(otap-dataflow): expose startup helpers as a public library module`

**Author:** Lalit Kumar Bhasin | **Date:** Apr 6, 2026 | **PR:** #2505 | **Fixes:** #2462, #2463

## Purpose

Makes the OTAP dataflow engine embeddable as a library so custom distribution binaries can reuse startup logic instead of copy-pasting from `main.rs`.

## Changes (7 files, +761 / -413)

| File | Change |
|---|---|
| `crates/controller/src/startup.rs` | **New** — 444 lines. Core of the commit. |
| `crates/controller/src/lib.rs` | Extended `Controller` with observer API. |
| `examples/custom_collector.rs` | **New** — 170-line runnable example. |
| `src/main.rs` | Slimmed from ~550 to ~140 lines; delegates to `startup::*`. |
| `README.md` | +61 lines — new "Embedding in Custom Distributions" section. |
| `Cargo.toml` | Removed 1 dep from the binary crate (moved to controller). |
| `crates/controller/Cargo.toml` | Added `sysinfo` dependency. |

## Key architectural changes

### 1. New `otap_df_controller::startup` module

Extracts three reusable functions:

- `apply_cli_overrides()` — merges `--num-cores`, `--core-id-range`, `--http-admin-bind` into an `OtelDataflowSpec`
- `validate_engine_components()` / `validate_pipeline_components()` — checks every node URN is registered in the `PipelineFactory` and runs per-component config validation
- `system_info()` — builds a diagnostics string (CPU cores, memory, build mode, allocator, registered component URNs)

All functions are now generic over `PData` and take a `&PipelineFactory` parameter instead of using the global `OTAP_PIPELINE_FACTORY` static directly — making them testable and reusable.

### 2. Observer API on `Controller`

Two new methods:

- `run_forever_with_observer(cfg, callback)`
- `run_till_shutdown_with_observer(cfg, callback)`

These invoke a `FnOnce(ObservedStateHandle)` callback after the state store is initialized, giving embedders zero-overhead in-process access to pipeline health/liveness without HTTP.

### 3. `src/main.rs` becomes a thin wrapper

All logic is replaced by calls to `startup::*`. The `memory_allocator_name()` helper remains in `main.rs` since allocator selection is a binary-crate concern.

## Test coverage

All existing unit tests for `core_allocation_override`, `http_admin_bind_override`, `apply_cli_overrides_*`, and `validate_unknown_component_rejected` were **moved** from `src/main.rs` to `startup.rs` (not deleted). Three additional policy-resolution tests were added (`apply_cli_overrides_only_changes_global_resources_policy`, `cli_num_cores_not_shadowed_by_implicit_default_resources`). Tests in `main.rs` that remained are CLI-parsing tests only.

## Observations

- Clean separation of concerns: binary crate owns CLI parsing and allocator selection; library crate owns validation, overrides, and diagnostics.
- The `run_with_mode` internal method was made generic (`<F: FnOnce(ObservedStateHandle)>`) with `Option<F>` — avoids a trait-object allocation while keeping the non-observer call sites ergonomic via `None::<fn(ObservedStateHandle)>`.
- No behavioral change for existing users — the refactor is purely structural.
