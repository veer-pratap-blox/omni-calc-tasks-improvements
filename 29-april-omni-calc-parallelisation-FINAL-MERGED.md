# Final Corrected Merged Omni-Calc Performance / Parallelisation Jira Roadmap

## Reference

Source planning file:

```text
https://github.com/veer-pratap-blox/tasks/blob/main/29-april-omni-calc-parallelisation-FINAL.md
```

Code branch:

```text
https://github.com/BloxSoftware/Blox-Dev/tree/BLOX-2104-improve-preload-snapshot-py-rust-pass
```

Primary Omni-Calc Rust areas:

```text
modelAPI/omni-calc/src/engine/exec/executor.rs
modelAPI/omni-calc/src/engine/exec/context.rs
modelAPI/omni-calc/src/engine/exec/state.rs
modelAPI/omni-calc/src/engine/exec/preload.rs
modelAPI/omni-calc/src/engine/exec/formula_eval.rs
modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs
modelAPI/omni-calc/src/engine/exec/get_source_data/dim_loader.rs
modelAPI/omni-calc/src/engine/exec/node_alignment/join_path.rs
modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs
modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs
modelAPI/omni-calc/src/engine/exec/steps/calculation.rs
modelAPI/omni-calc/src/engine/exec/steps/sequential.rs
modelAPI/omni-calc/src/engine/integration/calc_plan.rs
modelAPI/omni-calc/src/python.rs
modelAPI/omni-calc/Cargo.toml
```

---

# Important Correction

Earlier mapping incorrectly described Source Issue 10 as:

```text
Cached parsed formula AST and dependency metadata
```

Correct Source Issue 10 is:

```text
Issue 10 — Expand and Normalize PreloadedMetadata for Worker-Safe Execution
```

So Source Issue 10 belongs with the scheduler / worker-safety foundation because complete normalized preload is required before worker-safe execution.

Corrected statement:

```text
Final Issue 3 merges:
- Source Issue 1 — Core scheduler foundation
- Source Issue 10 — Expand and Normalize PreloadedMetadata for Worker-Safe Execution
```

---

# Global Rule For All Issues

After preload, Omni-Calc Rust execution must use Rust-owned data.

Allowed PyO3 usage:

```text
Python binding boundary
CalcPlan extraction
explicit preload before Rust execution
returning result back to Python
```

Not allowed inside Rust execution hot path:

```text
Python<'_>
PyObject
metadata_cache.call_method1(...)
metadata_cache.getattr(...)
lazy metadata callback from node execution
lazy metadata callback from formula evaluation
lazy metadata callback from resolver / join logic
lazy metadata callback from Rayon workers
```

Correct execution direction:

```text
Python boundary
    ↓
Preload all required metadata into Rust-owned PreloadedMetadata
    ↓
Run Rust execution under py.allow_threads
    ↓
Use ExecutionSnapshot / PreloadedMetadata / PropertyCacheSnapshot
    ↓
No Python callback inside executor hot path
```

---

# Final Corrected Source Issue Mapping

```text
Source Issue 1  -> Final Issue 3
Source Issue 2  -> Final Issue 10
Source Issue 3  -> Final Issue 6
Source Issue 4  -> Final Issue 6
Source Issue 5  -> Final Issue 5
Source Issue 6  -> Final Issue 5
Source Issue 7  -> Final Issue 2
Source Issue 8  -> Final Issue 2
Source Issue 9  -> Final Issue 5
Source Issue 10 -> Final Issue 3
Source Issue 11 -> Final Issue 4
Source Issue 12 -> Final Issue 4
Source Issue 13 -> Final Issue 1
Source Issue 14 -> Final Issue 1
Source Issue 15 -> Final Issue 7
Source Issue 16 -> Final Issue 8
Source Issue 17 -> Final Issue 9
Source Issue 18 -> Final Issue 1
Source Issue 19 -> Final Issue 2
Source Issue 20 -> Final Issue 4
Source Issue 21 -> Final Issue 2
```

---

# Final Correct Implementation Order

```text
1. Final Issue 1 — Benchmarking, Performance Baseline, and Debug Hot-Path Cleanup

2. Final Issue 2 — Data Structure, Clone Reduction, Shared References, FormulaEvaluator Context, and Typed IDs Foundation

3. Final Issue 3 — Core Kahn-Style Scheduler Foundation + Complete PreloadedMetadata Worker-Safe Execution

4. Final Issue 4 — Actuals, Forecast, and Dimension Row Metadata Optimization

5. Final Issue 5 — Cross-Object Join, Lookup, Aggregation, and Join-Key Optimization

6. Final Issue 6 — Pre/Post Processing Parallelism

7. Final Issue 7 — Standalone Source Issue 15

8. Final Issue 8 — Standalone Source Issue 16

9. Final Issue 9 — Standalone Source Issue 17

10. Final Issue 10 — Configurable Rayon-Based Parallel Ready-Node Execution
```

---

# Why This Order Is Correct

```text
Benchmark first
    ↓
Know current runtime and memory baseline

Data structure / clone cleanup second
    ↓
Avoid making Rayon execution memory-bound

Scheduler + complete preload foundation third
    ↓
Build safe execution architecture and remove Python callback risk

Actuals / dimension row metadata fourth
    ↓
Reduce repeated row-key and forecast handling costs

Cross-object join optimization fifth
    ↓
Optimize resolver/join paths before parallel workers hit them heavily

Pre/post parallelism sixth
    ↓
Implement safer independent parallel wins

Standalone issues next
    ↓
Resolve remaining specific source issues before final parallel execution

Rayon ready-node execution last
    ↓
Only after correctness, preloaded-only execution, resolver safety, and serial Kahn parity are proven
```

---

# FINAL ISSUE 1

## Issue Type

Performance / Observability / Tech Debt

---

## Title

Add Omni-Calc Performance Benchmark Baseline and Remove Debug Hot-Path Overhead

---

## Source Issues Merged

```text
Source Issue 13 - Benchmark harness / performance baseline / regression gates
Source Issue 14 - Debug hot-path cleanup / remove expensive diagnostics from production paths
Source Issue 18 - Performance observability / benchmark reporting / regression tracking
```

---

## Summary

Create a reliable benchmark, profiling, and observability foundation for the Rust Omni-Calc engine before implementing major scheduler, data-structure, clone-reduction, preload, resolver, join, or Rayon parallelism changes.

This issue should establish a clear before/after baseline for the current serial executor and remove debug/logging paths that allocate, clone, or distort runtime performance.

This issue should be solved first because every later optimization depends on measurable proof.

The goal is to ensure future performance work can answer:

```text
What is slow today?
Which phase is slow?
How much did the change improve?
Did the change regress another model shape?
Did memory usage increase?
Is Rayon actually helping or only adding overhead?
```

---

## Background / Context

Current Omni-Calc execution has multiple performance-sensitive phases:

```text
metadata preload
connected dimension preload
input step processing
property loading
calculation step processing
sequential step processing
formula evaluation
formula evaluator setup
cross-object resolver updates
join path creation
lookup aggregation
actuals / forecast handling
RecordBatch materialization
final result build
warning collection
debug / tracing diagnostics
clone-heavy state movement
```

Before optimizing these areas, we need reliable benchmarks and clean hot paths.

Without this baseline, later refactors may:

```text
look faster in theory but not in practice
improve one model shape but regress another
increase memory pressure
hide clone overhead behind parallelism
make Rayon execution memory-bound
make debugging harder
introduce performance regressions without detection
```

This issue is not about optimizing the executor yet. It is about creating the measurement and cleanup foundation required before larger optimization work starts.

---

## Child Issue Coverage

### Source Issue 13 — Benchmark Harness / Performance Baseline / Regression Gates

This source issue is covered by adding a benchmark harness and repeatable performance baseline for the Rust Omni-Calc engine.

Carried-forward scope:

```text
benchmark full Omni-Calc execution
benchmark major execution phases separately
record before/after timings
capture model-size metadata
support small / medium / large model benchmarks
support synthetic and realistic benchmark scenarios
support manual or CI regression checks
provide metrics for later optimization issues
```

### Source Issue 14 — Debug Hot-Path Cleanup

This source issue is covered by removing or gating debug/logging paths that allocate, clone, or collect expensive diagnostic data inside hot execution paths.

Carried-forward scope:

```text
remove or gate hardcoded debug logs
avoid always-on warn!/debug! diagnostics in hot loops
avoid collecting sample vectors unless tracing is enabled
avoid collecting column names unless tracing is enabled
avoid building sample key/value strings unless tracing is enabled
avoid clone-heavy diagnostics in normal execution
measure logging overhead with tracing disabled vs enabled
```

### Source Issue 18 — Performance Observability / Reporting / Regression Tracking

This source issue is covered by making benchmark output usable for future engineering decisions.

Carried-forward scope:

```text
add phase-level timing output
add benchmark report format
record baseline numbers
record model shape metadata
track runtime and memory-sensitive metrics
make results comparable across optimization branches
enable manual or CI performance regression checks
make later issues prove measurable improvement
```

---

## Problem Statement

Current risks:

```text
1. No consistent benchmark baseline for Omni-Calc execution.
2. No clear phase-level runtime breakdown.
3. No benchmark for metadata preload.
4. No benchmark for bulk preload vs fallback PyO3 preload.
5. No benchmark for formula evaluation alone.
6. No benchmark for formula evaluator setup / clone-heavy context setup.
7. No benchmark for calculation dependency-resolution sub-phases.
8. No benchmark for resolver update/materialization.
9. No benchmark for cross-object join alignment.
10. No benchmark for lookup aggregation.
11. No benchmark for final RecordBatch creation.
12. No benchmark for connected dimension preload.
13. No benchmark for actuals row-key construction.
14. No benchmark for forecast membership checks.
15. No benchmark for column lookup / duplicate-check cost.
16. No benchmark for warning collection / warning cloning cost.
17. Debug logs and diagnostics may allocate in hot paths.
18. Hardcoded debug paths can distort production performance.
19. Future refactors can accidentally regress performance without being caught.
```

---

## Code Areas / Operations To Benchmark

Primary Rust areas:

```text
modelAPI/omni-calc/src/engine/exec/executor.rs
modelAPI/omni-calc/src/engine/exec/context.rs
modelAPI/omni-calc/src/engine/exec/state.rs
modelAPI/omni-calc/src/engine/exec/preload.rs
modelAPI/omni-calc/src/engine/exec/formula_eval.rs
modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs
modelAPI/omni-calc/src/engine/exec/get_source_data/dim_loader.rs
modelAPI/omni-calc/src/engine/exec/node_alignment/join_path.rs
modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs
modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs
modelAPI/omni-calc/src/engine/exec/steps/calculation.rs
modelAPI/omni-calc/src/engine/exec/steps/sequential.rs
modelAPI/omni-calc/src/engine/integration/calc_plan.rs
modelAPI/omni-calc/src/python.rs
modelAPI/omni-calc/Cargo.toml
```

Important operations to benchmark:

```text
runtime::execute / full executor runtime
serial calc_steps loop
preload_metadata
collect_metadata_needs
extract_dimension_items_bulk
extract_property_rows
input value loading
input JSON parsing
forecast start filtering
actuals merge with input values
property loading from PreloadedMetadata
property cache population
calculation step processing
calculation dependency resolution
cross-object dependency resolution
dimension property column collection
connected dimension collection
formula evaluator setup
formula evaluation
formula AST / parsed_formula setup
PropertyFilterContext creation
time_values clone/setup
sequential step processing
resolver update
RecordBatch materialization
resolver RecordBatch extraction helpers
join path creation
lookup map creation
target alignment
lookup aggregation
warning collection
column lookup / duplicate checks
debug/tracing overhead
```

---

## Scope

Add benchmark coverage for the following major phases:

```text
1. Full Omni-Calc execution
2. Current serial calc_steps execution
3. Future single-threaded Kahn scheduler
4. Future Rayon ready-node execution
5. Metadata preload
6. Input indicator processing
7. Property loading from PreloadedMetadata
8. Formula evaluation
9. FormulaEvaluator setup
10. Calculation step processing
11. Sequential step processing
12. Resolver update/materialization
13. Cross-object join path creation
14. Lookup map aggregation
15. Final RecordBatch materialization
16. Connected dimension preload
17. Actuals / forecast handling
18. Clone-heavy paths
19. Memory allocation / peak memory where practical
```

Add explicit benchmark coverage for missing detailed sub-paths:

```text
20. PreloadedMetadata bulk preload vs fallback PyO3 preload path
21. collect_metadata_needs(plan) cost
22. preload extraction/parsing cost
23. extract_dimension_items_bulk cost
24. extract_property_rows cost
25. Calculation dependency-resolution sub-phases:
    - connected dimension collection for cross-object joins
    - cross-object dependency resolution
    - dimension property column collection
    - connected dimension property collection
26. Formula parsing / parsed_formula AST setup cost
27. PropertyFilterContext creation cost
28. time_values clone/setup cost
29. Actuals row-key construction:
    - build_dimension_key
    - build_entity_key
30. Forecast membership checks:
    - forecast_indices.contains
    - future forecast mask/bitset checks
31. Resolver RecordBatch extraction helpers:
    - get_indicator_values
    - get_dimension_columns
    - get_property_column
    - extract_string_columns_from_batch
    - extract_connected_dim_columns_from_batch
32. Warning collection and warning clone cost
33. HashMap / column lookup cost across:
    - state.number_columns
    - state.string_columns
    - state.connected_dim_columns
34. Duplicate-check cost during column merge/materialization
35. Debug/tracing overhead with logging disabled vs enabled
```

---

## Debug / Hot-Path Cleanup Scope

Clean up or gate the following:

```text
1. Hardcoded block-specific debug logs.
2. Hardcoded node-specific debug logs.
3. Always-on warn! logs in hot execution loops.
4. Debug-only vector sampling in hot paths.
5. Repeated column-name collection for logging.
6. Sample key/value construction unless tracing is enabled.
7. Clone-heavy diagnostics.
8. Expensive format! calls when logs are disabled.
9. Collecting first-N sample paths unless trace/debug is enabled.
10. Collecting available NodeMap keys unless trace/debug is enabled.
11. Collecting available filter keys unless trace/debug is enabled.
12. Summing large vectors only for logs unless trace/debug is enabled.
13. Building debug-only Vec<String> or Vec<&str> in normal execution.
```

Logging should follow this principle:

```text
If debug/tracing is disabled, diagnostic collection should have near-zero runtime and allocation overhead.
```

---

## Proposed Change

Add a benchmark harness or internal performance mode.

Possible implementation options:

```text
Rust criterion benchmarks
cargo bench benchmarks
internal benchmark command
test-only benchmark utilities
feature-gated benchmark mode
structured timing output from executor
```

Recommended benchmark structure:

```text
benches/
  omni_calc_full_execution.rs
  omni_calc_preload.rs
  omni_calc_input_steps.rs
  omni_calc_formula_eval.rs
  omni_calc_resolver.rs
  omni_calc_join_alignment.rs
  omni_calc_recordbatch.rs
  omni_calc_actuals.rs
```

Recommended performance instrumentation:

```text
PerfTimer / PhaseTimer helper
ExecutionTiming struct
BenchmarkResult struct
optional JSON output
optional CSV output
```

Example structure:

```rust
struct ExecutionTiming {
    total_runtime_ms: f64,
    preload_runtime_ms: f64,
    connected_dimension_preload_ms: f64,
    input_step_ms: f64,
    property_step_ms: f64,
    calculation_step_ms: f64,
    sequential_step_ms: f64,
    formula_eval_ms: f64,
    formula_setup_ms: f64,
    resolver_update_ms: f64,
    recordbatch_materialization_ms: f64,
    join_path_creation_ms: f64,
    lookup_aggregation_ms: f64,
    actuals_handling_ms: f64,
    clone_hotspot_ms: f64,
}
```

---

## Recommended Metrics

Add these high-level metrics:

```text
total_runtime_ms
preload_runtime_ms
connected_dimension_preload_ms
input_step_ms
property_step_ms
calculation_step_ms
sequential_step_ms
formula_eval_ms
formula_setup_ms
resolver_update_ms
recordbatch_materialization_ms
join_path_creation_ms
lookup_aggregation_ms
actuals_handling_ms
clone_hotspot_ms
row_count
block_count
column_count
node_count
warning_count
```

Add these detailed metrics:

```text
preload_bulk_ms
preload_fallback_ms
metadata_needs_ms
property_extract_ms
dimension_extract_ms
calc_dependency_resolution_ms
cross_object_resolution_ms
dimension_property_collection_ms
connected_dim_collection_ms
formula_parse_or_ast_setup_ms
property_filter_context_ms
time_values_clone_ms
actuals_row_key_ms
forecast_membership_ms
resolver_batch_extract_ms
warning_collection_ms
column_lookup_ms
duplicate_check_ms
debug_logging_overhead_ms
```

Add these memory/allocation-oriented metrics where practical:

```text
estimated_allocated_bytes
peak_memory_mb
number_column_clone_count
string_column_clone_count
recordbatch_clone_count
formula_context_clone_count
warning_clone_count
debug_vector_allocation_count
```

---

## Benchmark Scenarios

Add benchmark scenarios for:

```text
small model
medium model
large wide model
large row-count model
many-block model
cross-object-heavy model
actuals-heavy model
sequential-heavy model
property-heavy model
connected-dimension-heavy model
formula-heavy model
lookup-aggregation-heavy model
RecordBatch-heavy output model
debug-disabled run
debug-enabled run
```

Each benchmark result should record model shape:

```text
block_count
node_count
row_count_per_block
total_row_count
dimension_count
indicator_count
property_node_count
cross_object_reference_count
sequential_node_count
input_node_count
calculation_node_count
```

---

## Suggested Flags / Configuration

Add or support flags such as:

```text
CALC_DEBUG=false by default
CALC_PERF_TRACE=false by default
CALC_BENCH_MODE=true only in benchmark runs
CALC_LOG_HOT_PATH_DETAILS=false by default
```

Expected behavior:

```text
normal execution should not collect expensive debug data
benchmark mode can collect structured phase timings
debug mode can collect detailed diagnostics intentionally
```

---

## Expected Impact

```text
1. Makes future optimization measurable.
2. Prevents performance regressions.
3. Helps decide where Rayon is actually useful.
4. Helps identify clone-heavy and allocation-heavy paths.
5. Helps identify whether the bottleneck is execution, preload, resolver, joins, actuals, or output materialization.
6. Provides safe baseline before scheduler changes.
7. Makes it easier to prove improvements from Final Issue 2 through Final Issue 10.
8. Removes misleading debug overhead from production performance.
```

---

## Acceptance Criteria

```text
1. Benchmark harness exists.
2. At least small, medium, and large model benchmarks exist.
3. Benchmarks can compare serial vs optimized paths.
4. Benchmarks can compare debug-disabled vs debug-enabled execution.
5. Phase-level timing is available.
6. Detailed timing is available for preload, resolver, formula, actuals, join, and RecordBatch paths.
7. PreloadedMetadata bulk vs fallback PyO3 preload timing is available.
8. Calculation dependency-resolution timing is available.
9. Actuals row-key and forecast membership timing is available.
10. Resolver RecordBatch extraction timing is available.
11. Column lookup / duplicate-check timing is available.
12. Warning collection / warning clone timing is available.
13. Debug/tracing overhead timing is available.
14. Debug/hot-path logs do not allocate unless enabled.
15. Hardcoded block/node debug logs are removed or gated.
16. Baseline numbers are documented before other optimization work begins.
17. Performance regression checks can be added to CI or run manually.
18. Benchmark results can be used by later issues to prove improvement.
```

---

## Testing Notes

Add tests / benchmark cases for:

```text
full serial execution
future Kahn scheduler execution
future Rayon execution
large formula evaluation
formula evaluator setup
wide block column lookup
duplicate column check
large cross-object join
lookup aggregation
RecordBatch output
connected dimension preload
PreloadedMetadata bulk preload
PreloadedMetadata fallback preload
property extraction
actuals-heavy block
actuals row-key construction
forecast membership check
sequential-heavy block
resolver batch extraction
warning collection
debug disabled path
debug enabled path
```

---

## Out of Scope

```text
Changing calculation semantics
Adding Rayon ready-node execution
Changing output format
Changing Python DAG manager behavior
Refactoring scheduler
Changing resolver logic
Optimizing data structures
Reducing clones
Changing PreloadedMetadata shape
Changing formula evaluation behavior
Changing actuals behavior
Changing join semantics
```

This issue is measurement and cleanup only.

Implementation changes for optimization should happen in later final issues.

---

## Priority

Highest

---

## Dependencies

```text
None
```

This issue should be completed before:

```text
Final Issue 2 - Data Structure, Clone Reduction, Shared References, FormulaEvaluator Context, and Typed IDs Foundation
Final Issue 3 - Core Kahn-Style Scheduler Foundation + Worker-Safe PreloadedMetadata
Final Issue 4 - Actuals, Forecast, and Dimension Row Metadata Optimization
Final Issue 5 - Cross-Object Join, Lookup, Aggregation, and Join-Key Optimization
Final Issue 6 - Pre/Post Processing Parallelism
Final Issue 10 - Configurable Rayon-Based Parallel Ready-Node Execution
```

---

## Labels

```text
omni-calc
rust
benchmark
performance
profiling
debug-cleanup
regression-gates
observability
phase-timing
hot-path-cleanup
memory-profiling
```

---

## Components

```text
Omni-Calc
Rust Engine
Benchmarking
Performance
Debugging
Observability
Executor
Formula Evaluation
Preload
Resolver
RecordBatch Materialization
```

---

## Final Notes

This issue is the required first step for the Omni-Calc performance roadmap.

It should answer:

```text
Where is time currently spent?
Where are allocations happening?
Which debug paths distort runtime?
Which phase should be optimized first?
Which later issue produced measurable improvement?
```

Once this issue is complete, all later issues should use its benchmark harness and baseline numbers to prove correctness and performance improvement.
---

# FINAL ISSUE 2

## Issue Type

Performance / Data Structure Refactor / Memory Optimization

---

## Title

Optimize Omni-Calc Data Structures, Clone Reduction, Shared References, FormulaEvaluator Context, and Typed IDs Foundation

---

## Source Issues Merged

```text
Source Issue 7
Source Issue 8
Source Issue 19
Source Issue 21
```

---

## Summary

Refactor Omni-Calc internal data structures so execution can reuse Rust-owned preloaded data, reduce cloning, improve column lookup, and prepare for efficient scheduler-based and Rayon-based execution.

This issue should improve:

```text
CalcObjectState column lookup
NodeOutput merge performance
FormulaEvaluator input setup
FormulaEvaluator child evaluator cloning
PreloadedMetadata reuse
shared column storage
typed internal IDs
string parsing overhead
memory pressure
```

---

## Background / Context

Current state and evaluator paths are string-heavy and clone-heavy.

Common structures:

```rust
Vec<(String, Vec<f64>)>
Vec<(String, Vec<String>)>
HashMap<String, Vec<f64>>
HashMap<String, Vec<String>>
```

Current `CalcObjectState` concept:

```rust
pub struct CalcObjectState {
    pub object_key: String,
    pub object_type: CalcObjectType,

    pub dim_columns: Vec<(String, Vec<String>)>,
    pub row_count: usize,

    pub number_columns: Vec<(String, Vec<f64>)>,
    pub string_columns: Vec<(String, Vec<String>)>,
    pub connected_dim_columns: Vec<(String, Vec<String>)>,

    pub node_ids: HashSet<i64>,
}
```

Current lookup style:

```rust
self.number_columns
    .iter()
    .find(|(name, _)| name == column_name)
```

This is linear.

For wide models, this becomes expensive.

---

## Problem Statement

Current problems:

```text
1. Column lookup is often O(number_of_columns).
2. Duplicate checks scan Vec entries.
3. FormulaEvaluator owns and clones large column vectors.
4. Child evaluators can clone full EvalContext.
5. build_record_batch clones column vectors into Arrow arrays.
6. Resolver updates may repeatedly rebuild cloned RecordBatches.
7. PreloadedMetadata maps may be copied instead of reused.
8. Internal identifiers are string encoded.
9. Repeated string parsing / formatting costs CPU and memory.
10. Future Rayon execution can become memory-bandwidth bound if cloning remains high.
```

---

## Scope

### 1. Introduce ColumnStore

Add indexed column storage that preserves deterministic order.

Possible first version:

```rust
struct ColumnStore<T> {
    columns: Vec<(String, Arc<[T]>)>,
    index: HashMap<String, usize>,
}
```

Possible later version with typed IDs:

```rust
struct ColumnStore<T> {
    columns: Vec<(ColumnId, Arc<[T]>)>,
    index: HashMap<ColumnId, usize>,
}
```

Required methods:

```rust
insert(...)
get(...)
contains(...)
iter(...)
len(...)
is_empty(...)
```

---

### 2. Use Shared Column References

Introduce shared immutable column aliases:

```rust
type SharedNumberColumn = Arc<[f64]>;
type SharedStringColumn = Arc<[String]>;
```

or easier first step:

```rust
type SharedNumberColumn = Arc<Vec<f64>>;
type SharedStringColumn = Arc<Vec<String>>;
```

Preferred long-term:

```text
Arc<[T]>
```

because it clearly represents immutable shared slice data.

---

### 3. Refactor FormulaEvaluator Context

Current evaluator-like shape:

```rust
HashMap<String, Vec<f64>>
HashMap<String, Vec<String>>
Vec<String>
```

Target:

```rust
HashMap<String, Arc<[f64]>>
HashMap<String, Arc<[String]>>
Arc<[String]>
```

Avoid full context clone in child evaluator creation.

Target shape:

```rust
struct FormulaEvaluator {
    ctx: Arc<EvalContext>,
    warnings: Vec<EvalWarning>,
    actuals_context: Option<Arc<ActualsContext>>,
    property_filter_context: PropertyFilterContext,
    prior_called: bool,
    last_result_is_integer: bool,
}
```

---

### 4. Reuse PreloadedMetadata

Store/pass preloaded metadata as shared read-only data:

```rust
Arc<PreloadedMetadata>
```

Avoid cloning maps like:

```rust
HashMap<i64, Vec<DimensionItem>>
HashMap<(i64, i64, i64), HashMap<i64, String>>
```

during execution.

---

### 5. Add Typed IDs

Introduce typed internal IDs:

```rust
struct BlockId(i64);
struct IndicatorId(i64);
struct DimensionId(i64);
struct PropertyId(i64);
struct NodeId(i64);
```

Introduce column identity:

```rust
enum ColumnId {
    Indicator(IndicatorId),
    Dimension(DimensionId),
    Property {
        dimension_id: DimensionId,
        property_id: PropertyId,
    },
    CrossObjectIndicator {
        block_id: BlockId,
        indicator_id: IndicatorId,
    },
    ConnectedDimension {
        source_dimension_id: DimensionId,
        linked_dimension_id: DimensionId,
    },
}
```

Keep external output names unchanged.

---

## Proposed Implementation Plan

```text
Phase 1 - Add ColumnStore<T> with stable order and indexed lookup.
Phase 2 - Convert dynamic number/string/connected columns to ColumnStore.
Phase 3 - Add shared column aliases and avoid vector cloning in snapshots/evaluators where safe.
Phase 4 - Refactor FormulaEvaluator to share immutable EvalContext.
Phase 5 - Use Arc<PreloadedMetadata> / immutable property cache snapshots.
Phase 6 - Introduce typed ID helpers and conversion functions.
Phase 7 - Gradually migrate graph/scheduler/resolver to typed IDs where beneficial.
```

---

## Expected Impact

```text
1. Faster column lookup.
2. Faster duplicate checks.
3. Less memory allocation.
4. Lower peak memory usage.
5. Faster FormulaEvaluator setup.
6. Cheaper child evaluator creation.
7. Better use of Rust-owned preloaded data.
8. Better foundation for safe Rayon execution.
```

---

## Acceptance Criteria

```text
1. Dynamic column lookup is indexed, not linear.
2. Column insertion order remains deterministic.
3. Duplicate checks are O(1) average time.
4. FormulaEvaluator can read input columns without cloning full vectors.
5. with_raw_properties() does not clone full EvalContext.
6. PreloadedMetadata is reused by reference/Arc.
7. Property caches can be read-only where possible.
8. Typed ID helpers exist for block, indicator, dimension, property, node, and column identity.
9. External output names remain unchanged.
10. No Python/PyO3 callbacks are added.
11. Serial executor output remains identical.
12. Benchmarks show reduced allocations or runtime on large/wide models.
```

---

## Testing Notes

Add tests for:

```text
ColumnStore insert/get
ColumnStore duplicate insert
ColumnStore stable order
CalcObjectState column lookup parity
RecordBatch schema order
FormulaEvaluator shared context behavior
with_raw_properties shared context behavior
PreloadedMetadata reference reuse
typed ID parse / to_external_name roundtrip
serial output parity
large column lookup benchmark
clone/allocation benchmark
```

---

## Out of Scope

```text
Changing output schema
Changing calculation semantics
Changing Python DAG manager behavior
Full Rayon execution
Full formula engine rewrite
Removing human-readable debug names
Changing formula syntax
```

---

## Priority

Highest / High

---

## Dependencies

```text
Depends on Final Issue 1 for benchmark baseline.
Should be completed before Final Issue 3 and Final Issue 10.
```

---

## Labels

```text
omni-calc
rust
performance
data-structure
column-store
clone-reduction
shared-columns
arc
formula-evaluator
typed-ids
preloaded-data
memory-optimization
```

---

## Components

```text
Omni-Calc
Rust Engine
Execution State
Formula Evaluation
Preload
Column Store
Typed IDs
```

---
# FINAL ISSUE 3

## Issue Type

Technical Task / Refactor

---

## Title

Refactor Rust Omni-Calc Executor Toward Safe Kahn-Style DAG Scheduling Foundation, Worker-Safe PreloadedMetadata, and Cached Formula Dependency Metadata

---

## Source Issues Merged

```text
Source Issue 1  - Core scheduler foundation
Source Issue 10 - Expand and Normalize PreloadedMetadata for Worker-Safe Execution
```

Additional performance/scheduler scope added into this issue:

```text
Cached parsed formula AST and dependency metadata
```

---

## Important Correction

Source Issue 10 is:

```text
Expand and Normalize PreloadedMetadata for Worker-Safe Execution
```

It is **not**:

```text
Cached parsed formula AST and dependency metadata
```

However, **cached parsed formula AST and dependency metadata should still be included in this final issue** because it directly supports Rust-side graph construction, dependency detection, and Kahn-style scheduling.

So this final issue covers:

```text
1. Core scheduler foundation
2. Worker-safe normalized PreloadedMetadata
3. Cached parsed formula AST and formula dependency metadata
```

This updated version extends the earlier Final Issue 3 content with the missing formula-cache/dependency-metadata scope. :contentReference[oaicite:0]{index=0}

---

## Summary

Build the safe Rust execution architecture required before true parallel execution.

The current Rust Omni-Calc executor processes Python-provided `calc_steps` sequentially and mutates a shared `ExecutionContext`.

Target flow:

```text
Current serial executor
    ↓
Complete normalized PreloadedMetadata
    ↓
Cached parsed formula AST + dependency metadata
    ↓
ExecutionSnapshot
    ↓
NodeOutput
    ↓
Central merge phase
    ↓
Explicit Rust-side ExecutionGraph
    ↓
Single-threaded Kahn scheduler
    ↓
Serial parity proven
    ↓
Ready for Rayon in later issue
```

This issue should **not** implement Rayon parallel execution yet. It should build the correct architecture first.

---

## Background / Context

The current Rust executor flow is mainly in:

```text
modelAPI/omni-calc/src/engine/exec/executor.rs
```

Current simplified execution flow:

```rust
pub fn execute(_engine: &mut Engine, plan: Plan) -> Result<CalcResult> {
    let start = Instant::now();

    let mut ctx = ExecutionContext::new(&plan, _engine.preloaded_metadata());

    preload_connected_dimensions(&mut ctx);

    for step in &plan.request.calc_steps {
        match step.calc_type.as_str() {
            "input" => process_input_step(&mut ctx, step),
            "calculation" => process_calculation_step(&mut ctx, step),
            "sequential" => process_sequential_step(&mut ctx, step),
            _ => {}
        }
    }

    build_execution_result(&ctx, start)
}
```

Current model:

```text
Python provides ordered calc_steps
Rust executes each step sequentially
Each step mutates ExecutionContext
Resolver is updated during node processing
Formula dependency information is not owned as a scheduler-ready Rust graph
```

Rust does not currently own an explicit ready queue / Kahn scheduler.

---

## Current Execution Limitations

Current executor limitations:

```text
1. Rust trusts Python calc_steps ordering.
2. Rust does not track indegree/outgoing dependencies.
3. Rust does not know which nodes are independently ready.
4. Node execution mutates ExecutionContext directly.
5. Resolver is updated during node processing.
6. PreloadedMetadata may not yet be normalized for all worker execution needs.
7. Property maps and dimension items must be complete before execution.
8. Worker-safe execution cannot call Python/PyO3.
9. Sequential groups must remain atomic.
10. Formula dependency metadata is not yet represented as reusable scheduler-ready Rust metadata.
11. Parsed formula AST / dependency extraction may be repeated or distributed across execution paths.
```

---

## Problem Statement

The current serial executor is safe because only one node executes at a time, but it is not structured for safe parallelism.

A future parallel-ready executor needs:

```text
immutable node inputs
per-node outputs
centralized shared-state mutation
explicit dependency graph
complete preloaded metadata
cached formula dependency metadata
safe resolver snapshots
deterministic output merge
single-threaded Kahn parity before Rayon
```

Without this refactor, adding parallelism would either be unsafe or would require a global lock around `ExecutionContext`, which would serialize execution and provide little real performance benefit.

---

## Scope

## 1. Complete and Normalize PreloadedMetadata

Current preload direction:

```rust
pub struct PreloadedMetadata {
    pub dimension_items: HashMap<i64, Vec<DimensionItem>>,
    pub property_maps: HashMap<(i64, i64, i64), HashMap<i64, String>>,
}
```

This issue should expand and normalize preload so worker execution has everything required without Python callbacks.

Required preload coverage:

```text
dimension items
dimension item IDs
property maps
numeric property values
string property values
linked dimension properties
property maps by scenario
metadata needed for property nodes
metadata needed for cross-object joins
metadata needed for connected dimensions
metadata needed for filters
metadata needed by formula evaluation
metadata needed by formula dependency extraction
metadata needed by ExecutionSnapshot creation
```

Preloaded data should be stored in Rust-owned immutable/shared structures:

```rust
Arc<PreloadedMetadata>
```

or equivalent.

Missing required preload data should fail early before execution:

```text
Do not lazy-load missing metadata from Python during execution.
```

---

## 2. Add Cached Parsed Formula AST and Dependency Metadata

This is the newly added missing scope.

The executor should avoid repeatedly parsing formulas or repeatedly deriving dependencies during execution.

Instead, formula parsing and dependency extraction should happen once during planning / graph build / scheduler preparation.

### Proposed types

```rust
struct PlannedFormula {
    node_id: NodeId,
    block_id: BlockId,
    parsed_ast: Arc<FormulaAst>,
    dependency_info: FormulaDependencyInfo,
}
```

```rust
struct FormulaDependencyInfo {
    direct_indicator_deps: Vec<NodeId>,
    cross_object_refs: Vec<CrossObjectRef>,
    property_refs: Vec<PropertyRef>,
    dimension_refs: Vec<DimensionId>,
    connected_dimension_refs: Vec<DimensionId>,
    variable_filter_refs: Vec<VariableFilterRef>,
    uses_actuals: bool,
    uses_sequential_functions: bool,
    sequential_functions: Vec<String>,
}
```

```rust
struct CrossObjectRef {
    source_block_id: BlockId,
    source_node_id: NodeId,
    variable_name: String,
}
```

```rust
struct PropertyRef {
    dimension_id: DimensionId,
    property_id: PropertyId,
    scenario_id: Option<i64>,
}
```

### Why this belongs here

The Kahn scheduler needs to know:

```text
which nodes depend on which indicators
which nodes depend on cross-object source blocks
which nodes depend on property metadata
which nodes require connected dimensions
which nodes are sequential / not parallel-safe
which nodes can be added to the ready queue
```

Formula AST and dependency metadata are therefore part of the scheduler foundation.

### Current problem this solves

Without cached formula dependency metadata:

```text
dependency extraction may be repeated
graph building may rely on fragile string scanning
formula parsing may happen too late
scheduler readiness may miss hidden dependencies
cross-object dependencies may remain implicit
sequential formulas may be misclassified
```

### Target behavior

During scheduler setup:

```text
for each formula node:
    parse formula once
    extract dependency metadata once
    store PlannedFormula
    use dependency metadata to build ExecutionGraph
    use parsed AST during execution
```

Execution should then use:

```rust
planned_formula.parsed_ast
```

instead of reparsing or rediscovering dependencies.

### Required dependency coverage

Dependency metadata should cover:

```text
same-block indicator dependencies
cross-block / cross-object indicator dependencies
property references
dimension references
connected dimension references
variable filters
node maps
actuals dependency flags
sequential function flags
```

### Benefits

```text
1. Faster graph construction after first parse.
2. Less repeated formula parsing.
3. Cleaner dependency graph construction.
4. More reliable Kahn scheduler readiness.
5. Easier detection of parallel-safe vs non-parallel-safe nodes.
6. Better diagnostics when dependencies are missing.
7. Better support for future Rayon execution.
```

---

## 3. Add ExecutionSnapshot

Add a read-only node input structure.

Example:

```rust
struct ExecutionSnapshot {
    block_key: String,
    block_spec: Arc<BlockSpec>,

    dim_columns: Arc<...>,
    number_columns: Arc<...>,
    string_columns: Arc<...>,
    connected_dim_columns: Arc<...>,

    resolver_snapshot: Arc<CrossObjectResolverSnapshot>,
    preloaded_metadata: Arc<PreloadedMetadata>,
    property_cache_snapshot: Arc<PropertyCacheSnapshot>,
    planned_formula: Option<Arc<PlannedFormula>>,
}
```

Purpose:

```text
ExecutionSnapshot = read-only worker/node input
```

A node should be executable using snapshot data without mutating global executor state.

---

## 4. Add NodeOutput

Node execution should return output instead of directly mutating `ExecutionContext`.

Example:

```rust
struct NodeOutput {
    block_key: String,

    number_columns: Vec<...>,
    string_columns: Vec<...>,
    connected_dim_columns: Vec<...>,

    warnings: Vec<CalcWarning>,
    nodes_calculated: usize,

    should_update_resolver: bool,
}
```

Purpose:

```text
NodeOutput = worker/node result
```

---

## 5. Add Central Merge Phase

Only the merge phase should mutate `ExecutionContext`.

Example:

```rust
fn merge_node_output(ctx: &mut ExecutionContext, output: NodeOutput) {
    // only place where shared state is updated
}
```

Target behavior:

```text
execute node
    ↓
return NodeOutput
    ↓
merge output in deterministic coordinator-owned phase
```

Do not mutate shared state during node calculation.

---

## 6. Add ExecutionGraph

Add explicit Rust-side graph representation.

```rust
struct ExecutionGraph {
    nodes: HashMap<NodeId, ExecNode>,
    outgoing: HashMap<NodeId, Vec<NodeId>>,
    indegree: HashMap<NodeId, usize>,
}
```

Node type:

```rust
enum ExecNodeType {
    InputIndicator,
    Property,
    Calculation,
    SequentialGroup,
}
```

Node metadata:

```rust
struct ExecNode {
    id: NodeId,
    block_id: BlockId,
    calc_type: ExecNodeType,
    deps: Vec<NodeId>,
    outputs: Vec<ColumnId>,
    parallel_safe: bool,
    planned_formula: Option<Arc<PlannedFormula>>,
}
```

Graph should be built from:

```text
CalcPlan.calc_steps
CalcPlan.blocks
BlockSpec.indicators
cached PlannedFormula dependency metadata
CalcPlan.node_maps
CalcPlan.variable_filters
CalcPlan.property_specs
PreloadedMetadata
```

---

## 7. Add Single-Threaded Kahn Scheduler

First implementation must remain single-threaded.

Pseudo-flow:

```text
build graph
find zero-indegree nodes
execute one ready node
return NodeOutput
merge output
update resolver if needed
decrement dependents
enqueue newly ready nodes
```

Example:

```rust
let mut ready = VecDeque::new();

for node in graph.nodes.values() {
    if graph.indegree[&node.id] == 0 {
        ready.push_back(node.id);
    }
}

while let Some(node_id) = ready.pop_front() {
    let node = graph.nodes[&node_id].clone();

    let snapshot = build_snapshot(&ctx, &node);
    let output = execute_node(snapshot, node);

    merge_node_output(&mut ctx, output);

    update_resolver_if_needed(&mut ctx, &node);

    for dep in graph.outgoing[&node_id].iter() {
        graph.indegree[dep] -= 1;

        if graph.indegree[dep] == 0 {
            ready.push_back(*dep);
        }
    }
}
```

No Rayon in this issue.

---

## Resolver Requirements

Resolver updates must be coordinator-owned.

Do not update resolver from worker/node execution.

Target:

```text
execute node
return NodeOutput
merge output
update resolver at safe dependency boundary
release dependents
```

Add read-only resolver snapshot concept:

```rust
struct CrossObjectResolverSnapshot {
    calculated_blocks: Arc<HashMap<BlockId, BlockData>>,
    node_maps: Arc<HashMap<NodeMapKey, PlannedNodeMap>>,
    variable_filters: Arc<HashMap<String, VariableFilter>>,
}
```

Resolver snapshot should be built only from merged/stable state.

A node should never observe partially merged block state.

---

## Sequential Group Requirements

Sequential steps must remain atomic.

Represent as:

```rust
ExecNodeType::SequentialGroup
```

with:

```rust
parallel_safe = false
```

Do not split sequential group nodes into independent parallel nodes.

Sequential formulas/functions should be detected through cached formula dependency metadata:

```text
uses_sequential_functions = true
sequential_functions = ["prior", "balance", "change", "rollfwd", ...]
```

This helps avoid accidentally marking sequential nodes as parallel-safe.

---

## Python / Preload Requirement

Worker-ready execution path must use:

```text
PreloadedMetadata
PropertyCacheSnapshot
ExecutionSnapshot
Rust-owned plan data
cached PlannedFormula metadata
```

It must not call:

```text
Python<'_>
PyObject
metadata_cache.call_method1(...)
metadata_cache.getattr(...)
```

If any formula dependency requires metadata not available in `PreloadedMetadata`, execution should fail before scheduling starts.

---

## Rejected Design

Do not solve this by wrapping the full execution context in a global lock.

Rejected:

```rust
Arc<Mutex<ExecutionContext>>
```

around full node execution.

Reason:

```text
This serializes expensive node execution behind one lock and does not provide meaningful parallelism.
```

Preferred:

```text
ExecutionSnapshot = read-only input
NodeOutput = result
merge_node_output = short deterministic mutation phase
```

---

## Expected Impact

```text
1. Creates safe foundation for future parallelism.
2. Removes Python callback risk from worker execution.
3. Makes preload completeness explicit and testable.
4. Enables deterministic merge and resolver update boundaries.
5. Enables single-threaded Kahn scheduler parity testing.
6. Prevents global-lock-based false parallelism.
7. Reduces repeated formula parsing/dependency extraction.
8. Improves graph construction correctness.
9. Makes parallel-safe vs non-parallel-safe node classification clearer.
```

---

## Acceptance Criteria

```text
1. PreloadedMetadata is expanded/normalized for worker-safe execution.
2. All metadata required by worker-ready execution is available before execution starts.
3. Missing required metadata fails early with clear diagnostics.
4. Formula parsing/dependency extraction is cached as PlannedFormula or equivalent.
5. Formula dependency metadata covers same-block, cross-object, property, dimension, filter, and sequential-function dependencies.
6. ExecutionGraph uses cached formula dependency metadata instead of ad-hoc repeated parsing/scanning.
7. ExecutionSnapshot exists.
8. NodeOutput exists.
9. Shared state mutation is centralized in merge.
10. ExecutionGraph exists.
11. Graph tracks dependencies, outgoing edges, and indegrees.
12. Single-threaded Kahn scheduler exists behind config/feature flag.
13. Single-threaded Kahn output matches current serial calc_steps output.
14. Resolver updates happen at safe boundaries.
15. Sequential groups are atomic.
16. No global Arc<Mutex<ExecutionContext>> production design.
17. No PyO3/Python callbacks in execution hot path.
```

---

## Testing Notes

Add tests for:

```text
PreloadedMetadata completeness
missing dimension items
missing property map
missing linked dimension property
missing filter metadata
preloaded-only mode rejects missing metadata

formula AST cache creation
formula dependency metadata extraction
same-block indicator dependency extraction
cross-object dependency extraction
property dependency extraction
dimension dependency extraction
variable filter dependency extraction
sequential function detection
parallel_safe flag classification

ExecutionSnapshot build
NodeOutput merge
duplicate column merge
input -> calculation dependency
intra-block formula dependency
cross-block dependency
property dependency
sequential group atomic node
cycle detection
topological order parity
single-threaded Kahn vs current serial output
resolver update boundary
```

---

## Out of Scope

```text
Rayon execution
Full parallel scheduler
Work stealing
Parallel sequential groups
Changing output format
Changing calculation semantics
Changing Python DAG manager behavior
Lazy Python metadata fallback
Parallel formula evaluation
Full formula engine rewrite
```

---

## Priority

Highest

---

## Dependencies

```text
Depends on Final Issue 1.
Strongly benefits from Final Issue 2.
Required before Final Issue 10.
```

---

## Labels

```text
omni-calc
rust
executor
kahn-scheduler
dag
dependency-graph
execution-snapshot
node-output
resolver
preload
preloaded-metadata
formula-ast
formula-dependency-metadata
python-ffi
worker-safe
parallelism-foundation
```

---

## Components

```text
Omni-Calc
Rust Engine
Executor
Scheduler
Preload
Formula Evaluation
Cross-Object Resolver
Python Boundary
```

---

## Final Notes

This issue is the core foundation for future Omni-Calc parallelism.

It should prove correctness in a single-threaded scheduler before any Rayon ready-node execution is added.

The correct order is:

```text
Complete normalized PreloadedMetadata
    ↓
Cache parsed formula AST + dependency metadata
    ↓
Build ExecutionSnapshot / NodeOutput / merge phase
    ↓
Build ExecutionGraph
    ↓
Run single-threaded Kahn scheduler
    ↓
Prove parity with current serial calc_steps executor
    ↓
Enable Rayon parallel execution in a later issue
```
---

# FINAL ISSUE 4

## Issue Type

Performance / Algorithm Optimization / Refactor

---

## Title

Optimize Actuals, Forecast, Dimension Row Metadata, and Entity Key Reuse

---

## Source Issues Merged

```text
Source Issue 11
Source Issue 12
Source Issue 20
```

---

## Summary

Optimize repeated row metadata work in Omni-Calc.

This includes:

```text
actuals handling
forecast index/mask handling
row dimension key construction
entity key construction
dimension-combination reuse
avoiding repeated string-heavy row metadata
```

This is important for actuals-heavy, sequential-heavy, and large row-count models.

---

## Problem Statement

Actuals-heavy and sequential-heavy models repeatedly build row keys and entity keys.

Current expensive patterns:

```text
1. Build dimension keys per row per indicator.
2. Build entity keys per row per actuals/sequential path.
3. Sort/join key parts repeatedly.
4. Use Vec::contains for forecast index membership.
5. Clone actual_values and forecast_indices into ActualsContext.
6. Store repeated dimension item names as strings.
7. Rebuild row metadata that could be shared per block.
```

---

## Proposed Change

Add per-block row metadata cache:

```rust
struct BlockRowMetadata {
    row_dimension_keys: Arc<[String]>,
    row_entity_keys: Arc<[String]>,
    forecast_mask: Arc<[bool]>,
    forecast_indices: Arc<[usize]>,
}
```

Longer-term compact version:

```rust
struct BlockRowMetadata {
    row_dimension_key_ids: Arc<[u64]>,
    row_entity_key_ids: Arc<[u64]>,
    forecast_mask: Arc<[bool]>,
    forecast_indices: Arc<[usize]>,
}
```

Update actuals context:

```rust
struct ActualsContext {
    actual_values: Arc<[f64]>,
    forecast_indices: Arc<[usize]>,
    forecast_mask: Arc<[bool]>,
    last_actual_by_entity: Arc<HashMap<String, f64>>,
    has_actuals: bool,
}
```

---

## Proposed Implementation Plan

```text
Phase 1 - Precompute forecast mask per block.
Phase 2 - Precompute row dimension keys once per block.
Phase 3 - Precompute row entity keys once per block, excluding time dimension.
Phase 4 - Update actuals loading to use cached row keys.
Phase 5 - Update sequential/last-actual logic to reuse entity keys.
Phase 6 - Avoid unnecessary cloning in ActualsContext.
Phase 7 - Optionally move from string keys to compact IDs after typed IDs / join-key optimizations are ready.
```

---

## Expected Impact

```text
1. Faster actuals loading.
2. Faster sequential function setup.
3. Faster last-actual-by-entity extraction.
4. Less repeated string allocation.
5. Less repeated sorting/joining of key parts.
6. Faster forecast membership checks.
7. Reduced memory churn.
```

---

## Acceptance Criteria

```text
1. Forecast membership is O(1) using mask/bitset/HashSet.
2. Row dimension keys are computed once per block.
3. Row entity keys are computed once per block.
4. Actuals loading reuses cached row keys.
5. Last-actual-by-entity extraction reuses cached entity keys.
6. ActualsContext avoids unnecessary vector clones.
7. Dimension combination metadata is reused across evaluator/resolver/snapshots where possible.
8. Existing actuals behavior remains unchanged.
9. Existing sequential formula behavior remains unchanged.
10. No Python callbacks are introduced.
```

---

## Testing Notes

Add tests for:

```text
forecast mask correctness
forecast indices correctness
dimension key parity
entity key parity
actuals matching parity
last actual by entity parity
monthly forecast start
yearly forecast start
missing time dimension
multi-dimension block
sequential formula parity
serial output parity
large actuals benchmark
```

---

## Out of Scope

```text
Changing actuals semantics
Changing forecast-start behavior
Changing Python parity behavior
Changing output format
Parallel execution itself
```

---

## Priority

High / Medium

---

## Dependencies

```text
Depends on Final Issue 1.
Benefits from Final Issue 2.
Should be completed before Final Issue 10 for actuals/sequential-heavy models.
```

---

## Labels

```text
omni-calc
rust
actuals
forecast
dimension-metadata
row-key-cache
entity-key-cache
performance
clone-reduction
```

---

## Components

```text
Omni-Calc
Rust Engine
Calculation Handler
Sequential Handler
Actuals Handling
Formula Evaluation
```

---

# FINAL ISSUE 5

## Issue Type

Performance / Algorithm Optimization

---

## Title

Optimize Cross-Object Join Path Creation, Lookup Aggregation, and Join-Key Representation

---

## Source Issues Merged

```text
Source Issue 5
Source Issue 6
Source Issue 9
```

---

## Summary

Optimize cross-object resolver and node alignment performance.

Current cross-object alignment uses string-heavy join paths and sequential lookup/aggregation logic.

This issue should improve:

```text
join-path creation
target alignment
lookup map creation
aggregation
join key representation
optional parallel execution for large row counts
```

---

## Problem Statement

Current join paths are string-heavy.

Example conceptual key:

```text
Alice|Sales|Jan
```

Problems:

```text
1. Many string allocations.
2. Many string clones.
3. HashMap<String, f64> can be memory-heavy.
4. Join paths are built per source/target row.
5. Alignment is sequential.
6. Aggregation is sequential.
7. Large cross-object models spend significant time in join logic.
```

---

## Proposed Change

### Step 1: Threshold-Based Parallel Join Path Creation

Use Rayon above a row-count threshold:

```rust
(0..row_count)
    .into_par_iter()
    .map(|row_idx| build_join_path(...))
    .collect()
```

### Step 2: Parallel Target Alignment

```rust
target_join_paths
    .par_iter()
    .map(|path| lookup.get(path).copied().unwrap_or(default_value))
    .collect()
```

### Step 3: Deterministic Parallel Aggregation

Use per-thread local maps:

```text
split rows into chunks
each worker builds local HashMap
merge local maps deterministically
```

Support:

```text
sum
mean
first
last
```

For first/last:

```text
track original row index
```

### Step 4: Better Join-Key Representation

Move from:

```rust
HashMap<String, f64>
```

toward:

```rust
HashMap<JoinKey, f64>
```

Possible forms:

```rust
struct JoinKey(Vec<u32>);
struct JoinKey64(u64);
struct JoinKey {
    item_ids: SmallVec<[u32; 4]>,
}
```

Start with a compatible wrapper and migrate gradually.

---

## Expected Impact

```text
1. Faster large cross-object joins.
2. Less string allocation.
3. Less memory pressure.
4. Faster resolver alignment.
5. Better use of Rayon where row counts justify it.
```

---

## Acceptance Criteria

```text
1. Join-path creation has threshold-based parallel path.
2. Target alignment has threshold-based parallel path.
3. Lookup aggregation supports deterministic parallel reduction.
4. first/last aggregation preserves original row-order semantics.
5. Join key representation reduces string allocation where possible.
6. Serial fallback remains available.
7. Output ordering matches current behavior.
8. Cross-object resolver output matches current behavior.
9. Benchmarks show improvement on large row-count joins.
10. No Python callbacks are introduced.
```

---

## Testing Notes

Add tests for:

```text
simple join path creation
multi-dimension join path creation
missing target keys
target alignment with default values
sum aggregation
mean aggregation
first aggregation
last aggregation
duplicate paths
large row benchmark
serial vs parallel parity
threshold fallback behavior
```

---

## Out of Scope

```text
Changing join semantics
Changing aggregation semantics
Changing output format
Full resolver rewrite
Parallel ready-node scheduler
```

---

## Priority

High / Medium

---

## Dependencies

```text
Depends on Final Issue 1.
Benefits from Final Issue 2.
Should be completed before Final Issue 10 if cross-object-heavy models are common.
```

---

## Labels

```text
omni-calc
rust
cross-object
resolver
join-path
lookup-aggregation
join-key
rayon
performance
```

---

## Components

```text
Omni-Calc
Rust Engine
Cross-Object Resolver
Node Alignment
Lookup Aggregation
```

---

# FINAL ISSUE 6

## Issue Type

Performance / Parallelism

---

## Title

Parallelize Safe Pre/Post Execution Processing: RecordBatch Materialization and Connected Dimension Preload

---

## Source Issues Merged

```text
Source Issue 3
Source Issue 4
```

---

## Summary

Parallelize safer independent phases that do not require full Kahn ready-node execution.

This includes:

```text
final RecordBatch materialization across blocks
connected dimension preload across blocks
```

These are safer than node-level execution because they are naturally per-block and can use compute/merge phases.

---

## Problem Statement

Two expensive phases are good candidates for safe parallelism:

```text
1. Final RecordBatch materialization
2. Connected dimension preload
```

RecordBatch materialization happens after execution is complete, so state should be read-only.

Connected dimension preload is mostly independent per block, but currently mixes computation with `ExecutionContext` mutation.

---

## Proposed Change

### RecordBatch Materialization

Parallelize per-block output creation:

```rust
let batches: Vec<(String, RecordBatch)> = ctx.calc_object_states
    .par_iter()
    .filter(|(block_key, _)| block_key.starts_with('b'))
    .filter_map(|(block_key, state)| {
        build_record_batch(state)
            .ok()
            .map(|batch| (block_key.clone(), batch))
    })
    .collect();
```

Then sort deterministically:

```rust
batches.sort_by(|a, b| a.0.cmp(&b.0));
```

### Connected Dimension Preload

Split into compute/merge:

```text
compute connected dimension columns per block
merge results into ExecutionContext
```

Parallel compute:

```rust
block_keys
    .par_iter()
    .map(|block_key| compute_connected_dims_for_block(...))
    .collect()
```

---

## Expected Impact

```text
1. Faster output creation for many-block models.
2. Faster connected dimension preload for many-block/many-dimension models.
3. Lower wall-clock time in pre/post execution stages.
4. Lower risk than node-level parallelism.
```

---

## Acceptance Criteria

```text
1. RecordBatch materialization can run per block in parallel.
2. Connected dimension preload can compute per block in parallel.
3. Both paths have deterministic merge order.
4. Both paths have serial fallback.
5. Output matches current serial behavior.
6. Memory pressure is measured.
7. No Python callbacks are introduced.
```

---

## Testing Notes

Add tests for:

```text
multiple block outputs
RecordBatch schema parity
numeric columns
string columns
connected dimension columns
block output ordering
connected dimension preload parity
duplicate connected dimension handling
serial vs parallel output parity
large block materialization benchmark
```

---

## Out of Scope

```text
Parallel ready-node execution
Changing RecordBatch schema
Changing connected dimension semantics
Changing calculation semantics
```

---

## Priority

Medium

---

## Dependencies

```text
Depends on Final Issue 1 for benchmark baseline.
Safer after Final Issue 2.
Can be implemented before Final Issue 10.
```

---

## Labels

```text
omni-calc
rust
recordbatch
connected-dimensions
preload
rayon
parallelism
performance
```

---

## Components

```text
Omni-Calc
Rust Engine
Result Materialization
Preload
Connected Dimensions
```

---

# FINAL ISSUE 7

## Issue Type

Standalone / Specialized Task

---

## Title

Standalone Source Issue 15 — Verify Scope and Implement Without Losing the Original Requirement

---

## Source Issue

```text
Source Issue 15
```

---

## Summary

Keep Source Issue 15 as a standalone task unless its exact description clearly belongs to one of the merged issues.

This issue must not be deleted or ignored.

If Source Issue 15 is about scheduler safety, sequential group safety, global lock rejection, or preloaded-only execution boundaries, then it can be merged into Final Issue 3.

Otherwise, keep it separate and implement it after the core scheduler foundation is complete.

---

## Merge Guidance

Merge into Final Issue 3 only if Source Issue 15 is about:

```text
sequential group safety
do not parallelize sequential groups
scheduler correctness
single-threaded Kahn parity
global Arc<Mutex<ExecutionContext>> rejection
resolver update boundary safety
preloaded-only execution hot path
```

Keep standalone if it is about:

```text
a specialized performance optimization
a localized correctness issue
a targeted refactor
a separate risk area
```

---

## Recommended Placement

```text
After Final Issue 3
Before Final Issue 10
```

---

## Scope

```text
1. Re-open Source Issue 15 from the source document.
2. Confirm exact title and description.
3. Decide whether it belongs to Final Issue 3 or remains standalone.
4. If standalone, implement it after scheduler foundation and before Rayon ready-node execution.
5. Add tests according to the confirmed scope.
```

---

## Acceptance Criteria

```text
1. Source Issue 15 is reviewed and not lost.
2. Final merge decision is documented.
3. If merged into Final Issue 3, its requirements are added there.
4. If standalone, a clear Jira task exists.
5. Any required tests are added.
6. No Python callbacks are introduced.
7. No calculation semantics are changed unless explicitly intended.
```

---

## Priority

Medium / High depending on confirmed scope

---

## Dependencies

```text
Likely depends on Final Issue 3.
Must be resolved before Final Issue 10 if scheduler/parallel-safety related.
```

---

## Labels

```text
omni-calc
rust
standalone
verification-needed
scheduler-safety
performance
```

---

## Components

```text
Omni-Calc
Rust Engine
Executor
```

---

# FINAL ISSUE 8

## Issue Type

Standalone / Specialized Task

---

## Title

Standalone Source Issue 16 — Verify Scope and Implement Without Losing the Original Requirement

---

## Source Issue

```text
Source Issue 16
```

---

## Summary

Keep Source Issue 16 as a standalone task unless its exact description clearly belongs to one of the merged issues.

This issue must not be deleted or ignored.

If Source Issue 16 is about clone reduction, shared references, ColumnStore, typed IDs, preloaded data reuse, resolver, join path, lookup aggregation, or join-key representation, then it can be merged into Final Issue 2 or Final Issue 5.

Otherwise, keep it standalone.

---

## Merge Guidance

Merge into Final Issue 2 if Source Issue 16 is about:

```text
clone reduction
shared references
ColumnStore
CalcObjectState structure
FormulaEvaluator clone reduction
typed IDs
PreloadedMetadata reuse
less copying
memory optimization
```

Merge into Final Issue 5 if Source Issue 16 is about:

```text
resolver
join path
lookup aggregation
cross-object join alignment
join-key representation
String join-path reduction
```

Keep standalone if it is about:

```text
a specialized optimization
a narrow algorithmic issue
a correctness issue not covered by other groups
```

---

## Recommended Placement

```text
After Final Issue 2
Before Final Issue 10
```

If join-related:

```text
After Final Issue 5
Before Final Issue 10
```

---

## Scope

```text
1. Re-open Source Issue 16 from the source document.
2. Confirm exact title and description.
3. Decide whether it belongs to Final Issue 2, Final Issue 5, or standalone.
4. If standalone, define concrete implementation and test plan.
5. Ensure any optimization uses Rust-owned preloaded data.
6. Ensure no PyO3 callback is introduced in execution hot path.
```

---

## Acceptance Criteria

```text
1. Source Issue 16 is reviewed and not lost.
2. Final merge decision is documented.
3. If merged, its requirements are added to the correct merged issue.
4. If standalone, a clear Jira task exists.
5. Tests are added according to the confirmed scope.
6. No Python callbacks are introduced.
7. Existing output behavior remains unchanged.
```

---

## Priority

Medium / High depending on confirmed scope

---

## Dependencies

```text
Likely depends on Final Issue 2 or Final Issue 5.
Must be resolved before Final Issue 10 if it affects worker hot path.
```

---

## Labels

```text
omni-calc
rust
standalone
verification-needed
data-structure
preload
resolver
performance
```

---

## Components

```text
Omni-Calc
Rust Engine
Execution State
Preload
Resolver
```

---

# FINAL ISSUE 9

## Issue Type

Standalone / Specialized Task

---

## Title

Standalone Source Issue 17 — Verify Scope and Implement Without Losing the Original Requirement

---

## Source Issue

```text
Source Issue 17
```

---

## Summary

Keep Source Issue 17 as a standalone task unless its exact description clearly belongs to one of the merged issues.

This issue must not be deleted or ignored.

If Source Issue 17 is about actuals, forecast indices, entity key reuse, dimension row metadata, dimension combinations, RecordBatch materialization, connected dimension preload, or pre/post execution parallelism, then it can be merged into Final Issue 4 or Final Issue 6.

Otherwise, keep it standalone.

---

## Merge Guidance

Merge into Final Issue 4 if Source Issue 17 is about:

```text
actuals
forecast indices
entity key reuse
dimension row metadata
dimension combinations
row key cache
sequential metadata reuse
```

Merge into Final Issue 6 if Source Issue 17 is about:

```text
RecordBatch materialization
connected dimension preload
pre/post execution parallelism
per-block compute/merge
safe parallel preprocessing
```

Keep standalone if it is about:

```text
a specialized performance task
a localized correctness issue
a narrow implementation concern
```

---

## Recommended Placement

If actuals/metadata-related:

```text
After Final Issue 4
Before Final Issue 10
```

If pre/post processing-related:

```text
After Final Issue 6
Before Final Issue 10
```

---

## Scope

```text
1. Re-open Source Issue 17 from the source document.
2. Confirm exact title and description.
3. Decide whether it belongs to Final Issue 4, Final Issue 6, or standalone.
4. If standalone, define implementation plan and test coverage.
5. Ensure no Python callback is added.
6. Preserve output and calculation semantics.
```

---

## Acceptance Criteria

```text
1. Source Issue 17 is reviewed and not lost.
2. Final merge decision is documented.
3. If merged, its requirements are added to the correct merged issue.
4. If standalone, a clear Jira task exists.
5. Tests are added according to the confirmed scope.
6. Existing output behavior remains unchanged.
7. No Python callbacks are introduced.
```

---

## Priority

Medium depending on confirmed scope

---

## Dependencies

```text
Likely depends on Final Issue 4 or Final Issue 6.
Must be resolved before Final Issue 10 if it affects parallel worker hot path.
```

---

## Labels

```text
omni-calc
rust
standalone
verification-needed
actuals
metadata
preload
recordbatch
performance
```

---

## Components

```text
Omni-Calc
Rust Engine
Actuals
Metadata
Pre/Post Processing
```

---

# FINAL ISSUE 10

## Issue Type

Epic / Story / Parallel Execution

---

## Title

Add Configurable Rayon-Based Parallel Execution for Independent Ready Nodes

---

## Source Issues Merged

```text
Source Issue 2
```

This issue includes the related parallel execution work originally discussed as:

```text
Parallelize independent input indicators
Parallelize property map population/property nodes
Parallelize independent calculation nodes
Configurable Rayon/thread-pool strategy
```

---

## Summary

After the foundation and data-structure work is complete, add actual parallel execution for independent ready nodes using Rayon.

This should be the final major implementation step, not the first.

The Rust scheduler should use the Kahn-style ready queue from Final Issue 3 and execute only safe independent nodes in parallel.

---

## Why This Must Be Last

Do not implement Rayon ready-node execution until:

```text
1. Benchmark baseline exists.
2. Clone-heavy structures are reduced.
3. PreloadedMetadata is complete and normalized for worker-safe execution.
4. ExecutionSnapshot exists.
5. NodeOutput exists.
6. Central merge phase exists.
7. ExecutionGraph exists.
8. Single-threaded Kahn scheduler matches serial output.
9. Resolver updates are safe.
10. Preloaded-only execution path is enforced.
11. Python callbacks are removed from execution hot path.
12. Sequential groups remain atomic.
```

---

## Scope

Parallelize only safe ready nodes:

```text
input indicator nodes
property nodes using PreloadedMetadata
independent calculation nodes
```

Do not parallelize:

```text
sequential groups
nodes with unresolved dependencies
nodes requiring Python callbacks
nodes requiring mutable resolver access during execution
nodes that depend on another node in the same parallel batch
```

---

## Proposed Flow

```rust
while !ready.is_empty() {
    let parallel_batch = collect_parallel_safe_ready_nodes(&ready);

    let outputs: Vec<NodeOutput> = parallel_batch
        .into_par_iter()
        .map(|node| {
            let snapshot = build_snapshot_readonly(&ctx, &node);
            execute_node(snapshot, node)
        })
        .collect();

    for output in stable_merge_order(outputs) {
        merge_node_output(&mut ctx, output);
    }

    update_ready_queue();
}
```

---

## Config Requirements

Add config:

```rust
pub struct EngineConfig {
    pub enable_parallel_execution: bool,
    pub parallel_threads: Option<usize>,
    pub parallel_node_threshold: usize,
    pub parallel_row_threshold: usize,
}
```

Rules:

```text
parallel mode disabled by default until parity is proven
serial fallback always available
custom Rayon thread pool should avoid oversubscription
nested Rayon should be avoided or controlled
parallel merge order must be deterministic
```

---

## Parallelism Targets

### 1. Independent input indicators

Input indicator nodes can run in parallel when they have no unresolved dependencies and each writes distinct output columns.

### 2. Property nodes

Property nodes can run in parallel only when they read from:

```text
PreloadedMetadata
PropertyCacheSnapshot
ExecutionSnapshot
```

No Python callback is allowed.

### 3. Independent calculation nodes

Calculation nodes can run in parallel when:

```text
all dependencies are merged
required resolver snapshot is available
required property columns are available
output columns are distinct
node is not sequential
node does not depend on another node in same batch
```

---

## Expected Impact

```text
1. Main execution speedup for wide models.
2. Better CPU utilization.
3. Parallel calculation of dependency-free nodes.
4. Better scaling after clone reduction and data-structure cleanup.
```

---

## Acceptance Criteria

```text
1. Parallel execution can be enabled/disabled.
2. Worker thread count can be configured.
3. Independent input nodes can run in parallel.
4. Property nodes can run in parallel using preloaded data only.
5. Independent calculation nodes can run in parallel.
6. Workers do not mutate ExecutionContext.
7. Workers do not mutate resolver.
8. Workers do not call Python/PyO3.
9. Merge order is deterministic.
10. Output matches single-threaded Kahn execution.
11. Output matches original serial execution.
12. Sequential groups are excluded.
13. Benchmarks prove speedup on wide models.
14. Serial fallback remains available.
```

---

## Testing Notes

Add tests for:

```text
serial vs parallel parity
different Rayon thread counts
parallel independent input indicators
parallel property nodes
parallel independent calculation nodes
dependent nodes not running too early
cross-block dependency ordering
deterministic warnings
deterministic output schema/order
parallel disabled fallback
missing preload rejected in parallel mode
sequential group exclusion
```

---

## Out of Scope

```text
Parallelizing sequential groups
Changing calculation semantics
Changing output format
Changing Python DAG manager behavior
Custom work-stealing scheduler
```

---

## Priority

High, but only after prerequisite issues are complete

---

## Dependencies

```text
Depends on Final Issue 1.
Depends on Final Issue 2.
Depends on Final Issue 3.
Strongly benefits from Final Issue 4.
Strongly benefits from Final Issue 5.
Can benefit from Final Issue 6.
Should wait until Final Issue 7, 8, and 9 are resolved or explicitly confirmed as not blocking.
```

---

## Labels

```text
omni-calc
rust
rayon
parallel-execution
ready-node-execution
scheduler
performance
preloaded-data
no-python-callbacks
```

---

## Components

```text
Omni-Calc
Rust Engine
Executor
Scheduler
Formula Evaluation
Input Execution
Property Loading
```

---

# Final Dependency Map

```text
Final Issue 1
  -> required before all performance work

Final Issue 2
  -> should happen before Final Issue 3, 4, 5, 6, and 10

Final Issue 3
  -> required before Final Issue 10
  -> useful before Final Issue 4, 5, and 6

Final Issue 4
  -> should happen before Final Issue 10 for actuals/sequential-heavy models

Final Issue 5
  -> should happen before Final Issue 10 for cross-object-heavy models

Final Issue 6
  -> can happen before Final Issue 10
  -> safer after Final Issue 2 and Final Issue 3

Final Issue 7
  -> place after Final Issue 3 unless merged into another issue

Final Issue 8
  -> place after Final Issue 2 or 5 depending on confirmed scope

Final Issue 9
  -> place after Final Issue 4 or 6 depending on confirmed scope

Final Issue 10
  -> final major implementation step
```

---

# Final Recommended Order

```text
1. Final Issue 1 — Benchmarking, Performance Baseline, and Debug Hot-Path Cleanup

2. Final Issue 2 — Data Structure, Clone Reduction, Shared References, FormulaEvaluator Context, and Typed IDs Foundation

3. Final Issue 3 — Core Kahn-Style Scheduler Foundation + Worker-Safe PreloadedMetadata

4. Final Issue 4 — Actuals, Forecast, and Dimension Row Metadata Optimization

5. Final Issue 5 — Cross-Object Join, Lookup, Aggregation, and Join-Key Optimization

6. Final Issue 6 — Pre/Post Processing Parallelism

7. Final Issue 7 — Standalone Source Issue 15

8. Final Issue 8 — Standalone Source Issue 16

9. Final Issue 9 — Standalone Source Issue 17

10. Final Issue 10 — Configurable Rayon-Based Parallel Ready-Node Execution
```

---

# Final Notes

This corrected merged roadmap keeps all 21 source issues covered while reducing implementation tracking to 10 final Jira issues.

The most important correction is:

```text
Source Issue 10 = Expand and Normalize PreloadedMetadata for Worker-Safe Execution
```

Therefore Source Issue 10 belongs in:

```text
Final Issue 3 — Core Kahn-Style Scheduler Foundation + Worker-Safe PreloadedMetadata
```

Do not start Rayon parallel ready-node execution first.

Correct implementation path:

```text
measure baseline
    ↓
reduce clones and improve data structures
    ↓
expand/normalize PreloadedMetadata
    ↓
build snapshot/output/merge foundation
    ↓
build single-threaded Kahn scheduler
    ↓
prove serial parity
    ↓
optimize actuals / row metadata / joins / pre-post phases
    ↓
resolve standalone issues
    ↓
enable controlled Rayon parallelism
```

This creates a safer and faster Omni-Calc engine without introducing race conditions, Python callback bottlenecks, nondeterministic output ordering, or global lock bottlenecks.
