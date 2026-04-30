# Add Omni-Calc Runtime Performance Tracing and Benchmark Baseline

## Issue Type
Task

## Summary
Add runtime performance tracing and benchmark coverage for `modelAPI/omni-calc` so we can measure current serial execution, identify hot paths, and establish a reliable baseline before making further performance or parallelization changes.

## Background
The current branch uses Rust-based omni-calc execution, but we do not yet have a complete and reliable way to measure where time is being spent across the execution flow. We also need benchmark coverage to validate performance changes over time and compare the cost of debug or tracing output.

During validation, a few code-structure issues were identified in `python.rs`, and `cargo check` is currently blocked because `Cargo.toml` references benchmark files that do not yet exist. These issues need to be resolved as part of making performance instrumentation and benchmarking usable in this branch.

## Problem Statement
We currently lack:
- Structured runtime timing for key omni-calc phases
- A stable JSON timing output that can be consumed from Python
- Reproducible Criterion benchmarks for current serial execution
- Clean benchmark wiring in `Cargo.toml`
- Proper gating for hot-path debug logs so measurements are not distorted

Without this baseline, it is difficult to make safe optimization decisions or evaluate future Rust execution improvements.

## Goals
- Add runtime performance tracing behind environment flags
- Expose timing output through Python in a consumable JSON format
- Add benchmark coverage for current serial omni-calc execution and major hot paths
- Ensure benchmark setup compiles and runs successfully
- Remove or gate noisy debug logging that affects measurement quality
- Document how to run, save, and compare benchmarks

## Scope

### Runtime Tracing
Add structured timing coverage for major execution phases.

#### Track timings for:
- Total runtime
- Metadata preload
- Metadata needs collection
- Preload bulk calls
- Preload fallback paths
- Dimension/property extraction
- Serial calc steps
- Connected dimension preload
- Input/property/calculation/sequential steps
- Dependency and cross-object resolution
- Formula parse/eval/setup
- Actuals handling
- Resolver update and batch extraction
- Join path creation
- Lookup aggregation
- Target alignment
- RecordBatch materialization
- Final result build
- Warning collection
- Duplicate checks
- Clone counts / estimated clone bytes

### Python Exposure
Update the Python bridge so timing data can be retrieved through:
- `result.get_perf_timings_json()`

### Benchmarking
Add Criterion benchmark coverage for:
- Full current serial execution
- Small, medium, and large representative plans
- Formula evaluation hot paths
- Resolver/join alignment work
- RecordBatch creation
- Actuals-related paths
- Debug/tracing overhead comparisons

### Stability Fixes
As part of this work:
- Fix incorrect debug timing usage in `python.rs`
- Ensure logging references remain valid after config ownership changes
- Resolve `Cargo.toml` benchmark target issues by adding missing bench files or aligning benchmark entries with existing files

### Documentation
Add benchmark documentation covering:
- Setup and commands
- Baseline naming
- Comparison workflow
- Expected output artifacts
- Known limitations
- Placeholders for future scheduler/parallel benchmark scenarios

## Out of Scope
- Kahn scheduler implementation
- Rayon-based execution changes
- Future parallel DAG execution
- Broader backend refactors outside omni-calc performance tracing and benchmark enablement

## Acceptance Criteria
- `cargo check` runs successfully for `modelAPI/omni-calc`
- Benchmark target configuration is valid and no longer references missing files
- Runtime timing can be enabled through environment configuration
- Timing JSON is available through `result.get_perf_timings_json()`
- Criterion benchmarks run successfully for the defined omni-calc benchmark suite
- Hot-path debug logging is gated so benchmark results are not polluted by unconditional diagnostics
- Benchmark documentation exists with commands for running, saving, and comparing baselines
- The runtime timing implementation includes all listed execution phases and measurement categories in this ticket

## Notes
This ticket is intended to establish a trustworthy performance baseline for the current serial omni-calc path. Future optimization or parallelization work should build on the tracing and benchmark outputs produced here rather than being implemented in the same scope.
