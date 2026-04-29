# Final Corrected Merged Omni-Calc Performance / Parallelisation Jira Roadmap

## Reference

Source planning file:

```text
https://github.com/veer-pratap-blox/tasks/blob/main/29-april-omni-calc-parallelisation-FINAL.md
```

Current merged file to correct:

```text
https://github.com/veer-pratap-blox/tasks/blob/main/29-april-omni-calc-parallelisation-FINAL-MERGED.md
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

The previous merged file incorrectly mapped some source issues.

Incorrect previous mapping:

```text
Source Issue 13 -> Benchmarking / regression gates
Source Issue 14 -> Debug hot-path cleanup
Source Issue 18 -> Performance observability
```

Correct source issue titles are:

```text
Source Issue 13 -> Replace Hot-Path String Lookups with Interned IDs
Source Issue 14 -> Add Dependency-Aware Resolver Snapshot Refresh
Source Issue 18 -> Optimize Formula Dependency Parsing and Graph Build
```

The actual benchmarking issue from the source file is:

```text
Source Issue 16 -> Add Execution Profiling and Benchmarks for Scheduler, Snapshot, Merge, and Resolver
```

Also:

```text
Source Issue 15 -> Batch NodeOutput Merge Operations
Source Issue 17 -> Add Pre-Execution Validation for Parallel-Safety
```

So Source Issues 15, 16, and 17 should **not** remain vague standalone placeholder issues anymore. They have concrete source titles and should be merged into the correct final issues.

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

# Corrected Final Source Issue Mapping

```text
Source Issue 1  -> Final Issue 3
Source Issue 2  -> Final Issue 7
Source Issue 3  -> Final Issue 6
Source Issue 4  -> Final Issue 6
Source Issue 5  -> Final Issue 5
Source Issue 6  -> Final Issue 5
Source Issue 7  -> Final Issue 2
Source Issue 8  -> Final Issue 2
Source Issue 9  -> Final Issue 5
Source Issue 10 -> Final Issue 3
Source Issue 11 -> Final Issue 2
Source Issue 12 -> Final Issue 7
Source Issue 13 -> Final Issue 2
Source Issue 14 -> Final Issue 3
Source Issue 15 -> Final Issue 3
Source Issue 16 -> Final Issue 1
Source Issue 17 -> Final Issue 3
Source Issue 18 -> Final Issue 3
Source Issue 19 -> Final Issue 2
Source Issue 20 -> Final Issue 4
Source Issue 21 -> Final Issue 2
```

---

# Corrected Final Issue List

After correcting the source mappings, the roadmap should be reduced to **7 final merged issues**, not 10.

The previous 10-issue version kept Source Issues 15, 16, and 17 as standalone placeholders, but that is no longer correct because the source file has concrete titles for them.

Final corrected issues:

```text
Final Issue 1 - Add Omni-Calc Execution Profiling, Benchmarks, and Performance Regression Baseline

Final Issue 2 - Optimize Data Structures, Shared Column Storage, Clone Reduction, Interned IDs, FormulaEvaluator Context, and Typed IDs

Final Issue 3 - Refactor Executor Toward Kahn-Style Scheduler Foundation, Worker-Safe PreloadedMetadata, Dependency-Aware Resolver Refresh, Batched NodeOutput Merge, Parallel-Safety Validation, and Formula Dependency Graph Build

Final Issue 4 - Optimize Actuals, Forecast Indices, Entity Key Reuse, and Row Metadata

Final Issue 5 - Optimize Cross-Object Join Path Creation, Lookup Aggregation, and Join-Key Representation

Final Issue 6 - Parallelize Safe Pre/Post Processing: Final RecordBatch Materialization and Connected Dimension Preload

Final Issue 7 - Add Configurable Rayon-Based Ready-Layer / Ready-Node Parallel Execution
```

---

# Corrected Implementation Order

```text
1. Final Issue 1
   Add profiling, benchmarks, and performance baseline first.

2. Final Issue 2
   Fix data structures, clone-heavy paths, shared column storage, interned IDs, and typed IDs.

3. Final Issue 3
   Build safe scheduler foundation, complete worker-safe preload, resolver refresh rules, batched merge, safety validation, and formula dependency graph.

4. Final Issue 4
   Optimize actuals, forecast indices, entity keys, and row metadata.

5. Final Issue 5
   Optimize cross-object joins, lookup aggregation, and join-key representation.

6. Final Issue 6
   Parallelize safer pre/post execution work.

7. Final Issue 7
   Add Rayon-based ready-layer / ready-node execution last.
```

---

# Why This Order Is Correct

```text
Profiling first
    ↓
Know current runtime, memory, and phase-level baseline

Data structures / clone reduction second
    ↓
Avoid making Rayon execution memory-bound

Scheduler foundation third
    ↓
Build ExecutionSnapshot, NodeOutput, merge phase, graph, validation, and resolver refresh rules

Actuals / row metadata fourth
    ↓
Reduce repeated row-key and forecast handling cost

Cross-object join optimization fifth
    ↓
Optimize resolver/join paths before parallel workers hit them heavily

Pre/post parallelism sixth
    ↓
Implement safer independent parallel wins

Rayon ready-layer execution last
    ↓
Only after correctness, preload-only execution, resolver safety, and serial Kahn parity are proven
```

---

# FINAL ISSUE 1

## Issue Type

Performance / Observability / Tech Debt

---

## Title

Add Omni-Calc Execution Profiling, Benchmarks, and Performance Regression Baseline

---

## Source Issues Merged

```text
Source Issue 16 - Add Execution Profiling and Benchmarks for Scheduler, Snapshot, Merge, and Resolver
```

Additional code-analysis scope included:

```text
Debug hot-path cleanup
Benchmark result reporting
Performance regression tracking
Benchmark coverage for preload, formula, resolver, actuals, joins, RecordBatch, and clone-heavy paths
```

---

## Summary

Add benchmark and profiling coverage for the Rust Omni-Calc execution architecture to measure real performance gains and identify bottlenecks before implementing scheduler, snapshot, merge, resolver, data-structure, clone-reduction, or Rayon parallelism changes.

This issue should establish a repeatable baseline for:

```text
current serial executor
future single-threaded Kahn scheduler
future Rayon ready-layer execution
snapshot creation
node execution
NodeOutput merge
batched merge
resolver update
Arrow RecordBatch construction
metadata preload
formula dependency extraction
```

This issue should be solved first because every later optimization needs measurable proof.

---

## Background / Context

Executor refactors around snapshots, preloading, graph scheduling, resolver updates, Arrow `RecordBatch` construction, and Rayon parallelism need measurable validation.

Without benchmarks, it is difficult to know whether changes actually improve performance or only improve architecture.

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

---

## Problem Statement

Current risks:

```text
1. No consistent benchmark baseline for Omni-Calc execution.
2. No clear phase-level runtime breakdown.
3. No benchmark for graph construction.
4. No benchmark for formula dependency extraction.
5. No benchmark for metadata preload.
6. No benchmark for snapshot creation.
7. No benchmark for node execution.
8. No benchmark for NodeOutput merge.
9. No benchmark for batched merge.
10. No benchmark for resolver update.
11. No benchmark for Arrow RecordBatch construction.
12. No benchmark for serial Kahn scheduler.
13. No benchmark for parallel ready-layer execution.
14. No benchmark for large block execution.
15. No benchmark for cross-object dependency-heavy execution.
16. No benchmark for property-heavy execution.
17. No benchmark for sequential group execution.
18. Debug logs and diagnostics may allocate in hot paths.
19. Hardcoded debug paths can distort production performance.
20. Future refactors can accidentally regress performance without being caught.
```

---

## Scope

Add benchmarks for:

```text
1. Graph construction
2. Formula dependency extraction
3. Metadata preload
4. Snapshot creation
5. Node execution
6. NodeOutput merge
7. Batched merge
8. Resolver update
9. Arrow RecordBatch construction
10. Serial Kahn scheduler
11. Parallel ready-layer execution
12. Large block execution
13. Cross-object dependency-heavy execution
14. Property-heavy execution
15. Sequential group execution
```

Add broader benchmark coverage for:

```text
16. Full Omni-Calc execution
17. Current serial calc_steps execution
18. Connected dimension preload
19. Input indicator processing
20. Property loading from PreloadedMetadata
21. FormulaEvaluator setup
22. Cross-object join path creation
23. Lookup map aggregation
24. Actuals / forecast handling
25. Clone-heavy paths
26. Memory allocation / peak memory where practical
```

Add explicit detailed sub-path benchmark coverage for:

```text
27. PreloadedMetadata bulk preload vs fallback PyO3 preload path
28. collect_metadata_needs(plan) cost
29. extract_dimension_items_bulk cost
30. extract_property_rows cost
31. Calculation dependency-resolution sub-phases:
    - connected dimension collection for cross-object joins
    - cross-object dependency resolution
    - dimension property column collection
    - connected dimension property collection
32. Formula parsing / parsed_formula AST setup cost
33. PropertyFilterContext creation cost
34. time_values clone/setup cost
35. Actuals row-key construction:
    - build_dimension_key
    - build_entity_key
36. Forecast membership checks:
    - forecast_indices.contains
    - future forecast mask/bitset checks
37. Resolver RecordBatch extraction helpers:
    - get_indicator_values
    - get_dimension_columns
    - get_property_column
    - extract_string_columns_from_batch
    - extract_connected_dim_columns_from_batch
38. Warning collection and warning clone cost
39. HashMap / column lookup cost across:
    - state.number_columns
    - state.string_columns
    - state.connected_dim_columns
40. Duplicate-check cost during column merge/materialization
41. Debug/tracing overhead with logging disabled vs enabled
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

Logging rule:

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
  omni_calc_graph_build.rs
  omni_calc_snapshot.rs
  omni_calc_node_execution.rs
  omni_calc_node_output_merge.rs
  omni_calc_formula_eval.rs
  omni_calc_resolver.rs
  omni_calc_join_alignment.rs
  omni_calc_recordbatch.rs
  omni_calc_actuals.rs
  omni_calc_scheduler.rs
```

Recommended performance instrumentation:

```text
PerfTimer / PhaseTimer helper
ExecutionTiming struct
BenchmarkResult struct
optional JSON output
optional CSV output
```

---

## Recommended Metrics

High-level metrics:

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
graph_construction_ms
formula_dependency_extraction_ms
snapshot_creation_ms
node_execution_ms
node_output_merge_ms
batched_merge_ms
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

Detailed metrics:

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

Memory/allocation metrics where practical:

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

Suggested benchmark categories:

```text
Small plan:
  low number of blocks and indicators

Medium plan:
  moderate block count, formulas, properties, and joins

Large wide plan:
  many independent indicators and wide blocks

Large row-count plan:
  large row count per block

Cross-object-heavy plan:
  many block-to-block references and node maps

Property-heavy plan:
  many dimension properties and property columns

Sequential-heavy plan:
  many sequential groups and time-dependent formulas

Actuals-heavy plan:
  actual values, forecast boundaries, entity keys, and row-key lookups

RecordBatch-heavy plan:
  many blocks and large final output materialization

Debug-enabled vs debug-disabled plan:
  measure diagnostics overhead
```

Each result should record model shape:

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

## Acceptance Criteria

```text
1. Benchmark harness exists.
2. Benchmarks cover graph construction.
3. Benchmarks cover formula dependency extraction.
4. Benchmarks cover metadata preload.
5. Benchmarks cover snapshot creation.
6. Benchmarks cover node execution.
7. Benchmarks cover NodeOutput merge.
8. Benchmarks cover batched merge.
9. Benchmarks cover resolver update.
10. Benchmarks cover Arrow RecordBatch construction.
11. Benchmarks cover serial Kahn scheduler.
12. Benchmarks cover parallel ready-layer execution.
13. Benchmarks cover large block execution.
14. Benchmarks cover cross-object dependency-heavy execution.
15. Benchmarks cover property-heavy execution.
16. Benchmarks cover sequential group execution.
17. Benchmarks compare debug-disabled vs debug-enabled execution.
18. Phase-level timing is available.
19. Detailed timing is available for preload, resolver, formula, actuals, join, RecordBatch, snapshot, merge, and graph paths.
20. Baseline numbers are documented before other optimization work begins.
21. Benchmark results can be used by later issues to prove improvement.
22. Debug/hot-path logs do not allocate unless enabled.
23. Hardcoded block/node debug logs are removed or gated.
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

---

## Priority

Highest

---

## Dependencies

```text
None
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
Scheduler
```

---

# FINAL ISSUE 2

## Issue Type

Performance / Data Structure Refactor / Memory Optimization

---

## Title

Optimize Data Structures, Shared Column Storage, Clone Reduction, Interned IDs, FormulaEvaluator Context, and Typed IDs

---

## Source Issues Merged

```text
Source Issue 7  - Optimize CalcObjectState Column Storage and Lookup
Source Issue 8  - Reduce Cloning by Reusing Preloaded Data and Shared Column References
Source Issue 11 - Introduce Snapshot-Friendly Column Storage Using Shared Arc Data
Source Issue 13 - Replace Hot-Path String Lookups with Interned IDs
Source Issue 19 - Optimize FormulaEvaluator Context Cloning and Shared Column Access
Source Issue 21 - Introduce Structured Node and Column Identifiers to Reduce String Parsing and Lookup Cost
```

---

## Summary

Refactor Omni-Calc internal data structures so execution can reuse Rust-owned preloaded data, reduce cloning, improve column lookup, support snapshot-friendly shared storage, reduce hot-path string lookups, optimize FormulaEvaluator context cloning, and prepare for efficient scheduler-based and Rayon-based execution.

This issue should improve:

```text
CalcObjectState column lookup
NodeOutput merge performance
Snapshot-friendly column sharing
FormulaEvaluator input setup
FormulaEvaluator child evaluator cloning
PreloadedMetadata reuse
shared column storage
hot-path string lookup cost
interned IDs
typed internal IDs
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

Current problems include:

```text
linear column lookup
repeated duplicate checks
repeated string comparisons
large Vec clones
formula context cloning
RecordBatch clone-heavy materialization
string parsing in dependency/resolver paths
hot-path string formatting
```

---

## Scope

### 1. Optimize CalcObjectState Column Storage

Introduce indexed column storage that preserves deterministic order.

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

### 2. Introduce Snapshot-Friendly Shared Column Storage

Use immutable shared column storage so snapshots can cheaply share input data.

Recommended aliases:

```rust
type SharedNumberColumn = Arc<[f64]>;
type SharedStringColumn = Arc<[String]>;
```

or initially:

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

### 3. Reduce Cloning and Reuse Preloaded Data

Avoid unnecessary clones of:

```text
number columns
string columns
dimension columns
connected dimension columns
preloaded metadata maps
formula evaluator context
actuals context where possible
RecordBatch intermediate state
```

Store/pass preloaded metadata as shared read-only data:

```rust
Arc<PreloadedMetadata>
```

---

### 4. Optimize FormulaEvaluator Context Cloning

Move from:

```rust
HashMap<String, Vec<f64>>
HashMap<String, Vec<String>>
Vec<String>
```

toward:

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

### 5. Replace Hot-Path String Lookups With Interned IDs

Introduce an ID interning layer for frequently used strings such as:

```text
block keys
node IDs
column names
dimension column names
property column names
cross-object references
```

Example:

```rust
struct InternedId(u32);

struct StringInterner {
    strings: Vec<String>,
    index: HashMap<String, InternedId>,
}
```

Use interned IDs in hot paths while preserving external string names at output/debug boundaries.

---

### 6. Introduce Structured Node and Column Identifiers

Introduce typed IDs:

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
Phase 3 - Add Arc-backed SharedNumberColumn and SharedStringColumn.
Phase 4 - Make ExecutionSnapshot use shared column storage.
Phase 5 - Refactor FormulaEvaluator to share immutable EvalContext.
Phase 6 - Use Arc<PreloadedMetadata> / immutable property cache snapshots.
Phase 7 - Add string interner for hot-path names.
Phase 8 - Add typed ID helpers and conversion functions.
Phase 9 - Gradually migrate graph/scheduler/resolver to typed IDs where beneficial.
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
7. Cheaper snapshots.
8. Better use of Rust-owned preloaded data.
9. Fewer hot-path string comparisons.
10. Better foundation for safe Rayon execution.
```

---

## Acceptance Criteria

```text
1. Dynamic column lookup is indexed, not linear.
2. Column insertion order remains deterministic.
3. Duplicate checks are O(1) average time.
4. ExecutionSnapshot can share column data without deep clones.
5. FormulaEvaluator can read input columns without cloning full vectors.
6. with_raw_properties() does not clone full EvalContext.
7. PreloadedMetadata is reused by reference/Arc.
8. Interned IDs exist for selected hot-path string keys.
9. Typed ID helpers exist for block, indicator, dimension, property, node, and column identity.
10. External output names remain unchanged.
11. Debug/error messages remain readable.
12. No Python/PyO3 callbacks are added.
13. Serial executor output remains identical.
14. Benchmarks show reduced allocations or runtime on large/wide models.
```

---

## Testing Notes

Add tests for:

```text
ColumnStore insert/get
ColumnStore duplicate insert
ColumnStore stable order
CalcObjectState column lookup parity
snapshot shared column reuse
RecordBatch schema order
FormulaEvaluator shared context behavior
with_raw_properties shared context behavior
PreloadedMetadata reference reuse
string interner roundtrip
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
Replacing every string in one PR
```

---

## Priority

Highest / High

---

## Dependencies

```text
Depends on Final Issue 1 for benchmark baseline.
Should be completed before Final Issue 3 and Final Issue 7.
```

---

## Labels

```text
omni-calc
rust
performance
data-structure
column-store
shared-columns
arc
clone-reduction
formula-evaluator
interned-ids
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
String Interning
```

---

# FINAL ISSUE 3

## Issue Type

Technical Task / Refactor / Scheduler Foundation

---

## Title

Refactor Executor Toward Kahn-Style Scheduler Foundation, Worker-Safe PreloadedMetadata, Dependency-Aware Resolver Refresh, Batched NodeOutput Merge, Parallel-Safety Validation, and Formula Dependency Graph Build

---

## Source Issues Merged

```text
Source Issue 1  - Refactor Rust Omni-Calc Executor Toward Safe Kahn-Style DAG Scheduling Foundation
Source Issue 10 - Expand and Normalize PreloadedMetadata for Worker-Safe Execution
Source Issue 14 - Add Dependency-Aware Resolver Snapshot Refresh
Source Issue 15 - Batch NodeOutput Merge Operations
Source Issue 17 - Add Pre-Execution Validation for Parallel-Safety
Source Issue 18 - Optimize Formula Dependency Parsing and Graph Build
```

---

## Summary

Build the safe Rust execution architecture required before true parallel execution.

This issue merges all scheduler-foundation work:

```text
immutable ExecutionSnapshot
per-node NodeOutput
central merge phase
batched NodeOutput merge
explicit Rust-side ExecutionGraph
single-threaded Kahn scheduler
worker-safe normalized PreloadedMetadata
dependency-aware resolver snapshot refresh
pre-execution parallel-safety validation
formula dependency parsing and graph build optimization
```

This issue should **not** implement Rayon execution yet. It should build the correct architecture first.

---

## Background / Context

Current model:

```text
Python provides ordered calc_steps
Rust creates one mutable ExecutionContext
Rust executes each step sequentially
Each node mutates ExecutionContext directly
Resolver is updated during node processing
Formula dependencies are not fully represented as scheduler-ready Rust metadata
```

Target model:

```text
Before execution:
  Python builds CalcPlan
  Rust validates plan
  Rust preloads metadata
  Rust builds execution graph
  Rust validates parallel-safety

During execution:
  Coordinator owns ExecutionContext
  Coordinator owns ready queue and graph indegrees
  Workers receive immutable ExecutionSnapshot
  Workers execute node logic
  Workers return NodeOutput
  Merge phase mutates ExecutionContext

After merge:
  Resolver refresh happens only when required
  Dependent nodes are released
  Final output format remains unchanged
```

---

## Scope

### 1. Add ExecutionSnapshot

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

---

### 2. Add NodeOutput

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

---

### 3. Add Batched NodeOutput Merge

Improve merge performance by batching multiple `NodeOutput` values before mutating `ExecutionContext`.

Example:

```rust
fn merge_node_outputs(ctx: &mut ExecutionContext, outputs: Vec<NodeOutput>) {
    // group outputs by block_id
    // append columns in batch
    // collect warnings
    // update nodes_calculated
    // refresh resolver once per affected block when required
}
```

Group outputs by:

```text
block_id
affected output columns
resolver update requirement
warning collection
calculated node count
```

Acceptance for batching:

```text
Multiple NodeOutput values can be merged in one controlled mutation phase.
Outputs are grouped by block before merge.
Warnings and stats are accumulated once per batch.
Resolver refresh happens at most once per affected block per merge batch.
Existing serial behavior remains unchanged.
```

---

### 4. Add ExecutionGraph

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

---

### 5. Add Single-Threaded Kahn Scheduler

First implementation must remain single-threaded.

Pseudo-flow:

```text
build graph
find zero-indegree nodes
execute one ready node
return NodeOutput
merge output
refresh resolver if required
decrement dependents
enqueue newly ready nodes
```

No Rayon in this issue.

---

### 6. Expand and Normalize PreloadedMetadata

Complete and normalize preload so worker execution has everything required without Python callbacks.

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

Missing required preload data should fail before execution:

```text
Do not lazy-load missing metadata from Python during execution.
```

---

### 7. Add Dependency-Aware Resolver Snapshot Refresh

Resolver should not be blindly refreshed after every node output.

Only refresh resolver state when:

```text
A source block has completed the required columns for downstream references
A dependency boundary is reached
A ready batch/layer has finished
A downstream node is about to read cross-object data
Resolver-visible data has actually changed
```

Example concept:

```rust
struct ResolverRefreshPlan {
    required_by_block: HashMap<BlockId, Vec<CrossObjectDependency>>,
    dirty_blocks: HashSet<BlockId>,
}
```

Resolver updates should happen in a controlled merge phase, not inside each node execution.

Acceptance for resolver refresh:

```text
Resolver refresh is not blindly triggered after every node output.
Resolver update decisions are dependency-aware.
Cross-object references still resolve correctly.
Serial behavior remains unchanged.
Resolver is updated before downstream nodes that need source data are released.
Performance improves for plans with multiple outputs from the same source block.
```

---

### 8. Add Pre-Execution Validation for Parallel-Safety

Before execution, validate every planned node and classify whether it is parallel-safe.

Validation should check:

```text
Whether all metadata is preloaded
Whether dependencies are explicit
Whether outputs are distinct
Whether the node belongs to a sequential group
Whether it requires resolver updates before downstream execution
Whether it depends on Python callbacks
Whether it mutates shared state
Whether output columns overlap with another ready node
```

Acceptance for validation:

```text
Every node has a clear parallel-safety classification.
Unsafe nodes are excluded from Rayon execution.
Validation errors are clear and actionable.
Sequential nodes are always marked unsafe initially.
Missing metadata marks the node as unsafe or fails before execution.
```

---

### 9. Optimize Formula Dependency Parsing and Graph Build

Extract formula dependencies once during graph construction.

Store parsed dependency information directly in each `ExecNode`.

Example:

```rust
struct ExecNode {
    id: NodeId,
    block_id: BlockId,
    deps: Vec<NodeId>,
    output_columns: Vec<ColumnId>,
    same_block_deps: Vec<ColumnId>,
    cross_object_deps: Vec<CrossObjectDependency>,
    property_deps: Vec<PropertyId>,
    parallel_safe: bool,
}
```

Example cross-object dependency:

```rust
struct CrossObjectDependency {
    source_block_id: BlockId,
    source_column_id: ColumnId,
    required_before_node: NodeId,
}
```

The scheduler should use this dependency information directly instead of parsing formula strings repeatedly.

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
merge_node_outputs = short deterministic mutation phase
resolver refresh = dependency-aware coordinator action
```

---

## Expected Impact

```text
1. Creates safe foundation for future parallelism.
2. Removes Python callback risk from worker execution.
3. Makes preload completeness explicit and testable.
4. Enables deterministic merge and resolver update boundaries.
5. Avoids repeated resolver refresh after every node.
6. Enables batched merge for ready-layer execution.
7. Enables single-threaded Kahn scheduler parity testing.
8. Reduces repeated formula dependency parsing.
9. Improves graph construction correctness.
10. Prevents global-lock-based false parallelism.
```

---

## Acceptance Criteria

```text
1. ExecutionSnapshot exists.
2. NodeOutput exists.
3. Multiple NodeOutput values can be batched and merged deterministically.
4. Shared state mutation is centralized in merge.
5. ExecutionGraph exists.
6. Graph tracks dependencies, outgoing edges, and indegrees.
7. Sequential groups are represented as atomic non-parallel-safe nodes.
8. Single-threaded Kahn scheduler exists behind config/feature flag.
9. Single-threaded Kahn output matches current serial calc_steps output.
10. PreloadedMetadata is expanded/normalized for worker-safe execution.
11. Missing required metadata fails early with clear diagnostics.
12. Resolver refresh is dependency-aware.
13. Resolver update decisions happen in merge/coordinator phase.
14. Formula dependencies are parsed once during graph construction.
15. Same-block dependencies are represented explicitly.
16. Cross-object dependencies are represented explicitly.
17. Property dependencies are represented explicitly.
18. Scheduler does not parse formula strings repeatedly.
19. Every node has a clear parallel-safety classification.
20. Unsafe nodes are excluded from future Rayon execution.
21. No production implementation uses global Arc<Mutex<ExecutionContext>> around full node execution.
22. No PyO3/Python callbacks occur inside execution hot path after preload.
```

---

## Testing Notes

Add tests for:

```text
ExecutionSnapshot build
NodeOutput merge
batched NodeOutput merge
multiple outputs for same block
multiple outputs for different blocks
duplicate column prevention
warning merge behavior
resolver update behavior during batch merge
preloaded metadata completeness
missing metadata failure
input -> calculation dependency
same-block dependency extraction
cross-object dependency extraction
property dependency extraction
sequential group dependency extraction
formula dependency parsing
graph ordering
missing dependency detection
invalid formula reference errors
resolver refresh timing
missing source data errors
parallel-safe classification
unsafe node classification
overlapping output columns
Python callback dependency detection
sequential group safety handling
single-threaded Kahn vs current serial output
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
Required before Final Issue 7.
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
batched-merge
resolver
resolver-refresh
preload
preloaded-metadata
formula-dependency-metadata
parallel-safety
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

# FINAL ISSUE 4

## Issue Type

Performance / Algorithm Optimization / Refactor

---

## Title

Optimize Actuals, Forecast Indices, Entity Key Reuse, and Row Metadata

---

## Source Issues Merged

```text
Source Issue 20 - Optimize Actuals Handling, Forecast Indices, and Entity Key Reuse
```

---

## Summary

Optimize actuals and forecast-period handling in Omni-Calc by reducing repeated row-key construction, avoiding repeated string allocation, improving forecast membership checks, and reusing precomputed entity/dimension keys.

This is important for:

```text
actuals-heavy models
sequential-heavy models
large row-count models
models with many dimensions
models with many forecast/actual periods
```

---

## Problem Statement

Actuals-heavy and sequential-heavy models can pay the same key-building cost repeatedly.

Current expensive patterns:

```text
1. Dimension keys are rebuilt per row per actuals-enabled indicator.
2. Entity keys are rebuilt per row per indicator/function.
3. key_parts Vec is allocated repeatedly.
4. String format/join/sort happens repeatedly.
5. forecast_indices membership uses Vec::contains.
6. actual_values and forecast_indices are cloned into ActualsContext.
7. Existing column lookup may still use linear scans.
```

---

## Proposed Change

Introduce reusable per-block actuals/key metadata.

First version:

```rust
struct BlockRowKeyCache {
    row_dimension_keys: Arc<[String]>,
    row_entity_keys: Arc<[String]>,
    forecast_mask: Arc<[bool]>,
    forecast_indices: Arc<[usize]>,
}
```

Long-term version:

```rust
struct BlockRowKeyCache {
    row_dimension_key_ids: Arc<[u64]>,
    row_entity_key_ids: Arc<[u64]>,
    forecast_mask: Arc<[bool]>,
    forecast_indices: Arc<[usize]>,
}
```

Update `ActualsContext`:

```rust
pub struct ActualsContext {
    pub actual_values: Arc<[f64]>,
    pub forecast_indices: Arc<[usize]>,
    pub forecast_mask: Arc<[bool]>,
    pub last_actual_by_entity: Arc<HashMap<String, f64>>,
    pub has_actuals: bool,
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
6. O(1) forecast membership checks.
7. Reduced memory churn.
```

---

## Acceptance Criteria

```text
1. Forecast membership check is O(1).
2. Row dimension keys are computed once per block and reused.
3. Row entity keys are computed once per block and reused.
4. Actuals loading uses precomputed row keys.
5. Last-actual-by-entity extraction uses precomputed entity keys.
6. ActualsContext avoids unnecessary vector clones where possible.
7. Existing actuals behavior remains unchanged.
8. Existing sequential formula behavior remains unchanged.
9. No Python callbacks are introduced.
10. Benchmarks show improvement on actuals-heavy or sequential-heavy models.
```

---

## Testing Notes

Add tests for:

```text
forecast mask correctness
forecast indices correctness
dimension key parity with old build_dimension_key
entity key parity with old build_entity_key
actuals matching parity
last actual by entity parity
monthly forecast start
yearly forecast start
missing time dimension
multi-dimension block
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
Full structured ID migration
Parallel execution itself
Removing sequential group atomicity
```

---

## Priority

Medium / High

---

## Dependencies

```text
Depends on Final Issue 1.
Benefits from Final Issue 2.
Should be completed before Final Issue 7 for actuals/sequential-heavy models.
```

---

## Labels

```text
omni-calc
rust
actuals
forecast
performance
row-key-cache
entity-key-cache
clone-reduction
algorithm-optimization
no-python-callbacks
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
Source Issue 5 - Parallelize Join-Path Creation and Target Alignment for Large Cross-Object Joins
Source Issue 6 - Evaluate Parallel Lookup-Map Aggregation with Deterministic Reductions
Source Issue 9 - Optimize Join Keys and Avoid Repeated String Join Path Allocation
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
Should be completed before Final Issue 7 if cross-object-heavy models are common.
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

Parallelize Safe Pre/Post Execution Processing: Final RecordBatch Materialization and Connected Dimension Preload

---

## Source Issues Merged

```text
Source Issue 3 - Parallelize Final RecordBatch Materialization Across Blocks
Source Issue 4 - Parallelize Connected Dimension Preload Across Blocks
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
Can be implemented before Final Issue 7.
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

Epic / Story / Parallel Execution

---

## Title

Add Configurable Rayon-Based Ready-Layer / Ready-Node Parallel Execution

---

## Source Issues Merged

```text
Source Issue 2  - Add Configurable Rayon-Based Parallel Execution for Independent Ready Nodes
Source Issue 12 - Add Rayon-Based Ready-Layer Parallel Execution Behind a Feature Flag
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
6. Batched merge exists.
7. ExecutionGraph exists.
8. Single-threaded Kahn scheduler matches serial output.
9. Resolver refresh is dependency-aware.
10. Pre-execution validation classifies parallel-safe nodes.
11. Preloaded-only execution path is enforced.
12. Python callbacks are removed from execution hot path.
13. Sequential groups remain atomic.
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
nodes with overlapping output columns
```

---

## Proposed Flow

```rust
while !ready.is_empty() {
    let ready_layer = collect_parallel_safe_ready_layer(&ready);

    let outputs: Vec<NodeOutput> = ready_layer
        .into_par_iter()
        .map(|node| {
            let snapshot = build_snapshot_readonly(&ctx, &node);
            execute_node(snapshot, node)
        })
        .collect();

    merge_node_outputs(&mut ctx, stable_merge_order(outputs));

    refresh_resolver_if_required(&mut ctx);

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
feature flag should allow safe rollout
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

## Why Rayon Is Appropriate

Rayon is appropriate because this is CPU-bound Rust work:

```text
formula evaluation
input value generation
property map processing
vector calculations
dependency-free ready-layer execution
```

Async is not appropriate for these execution paths because there is no IO wait inside worker node execution after preload.

Custom work-stealing should not be implemented initially. Rayon’s default scheduler is sufficient for the first implementation.

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
3. Ready-layer parallel execution is behind a feature flag.
4. Independent input nodes can run in parallel.
5. Property nodes can run in parallel using preloaded data only.
6. Independent calculation nodes can run in parallel.
7. Workers do not mutate ExecutionContext.
8. Workers do not mutate resolver.
9. Workers do not call Python/PyO3.
10. Merge order is deterministic.
11. Output matches single-threaded Kahn execution.
12. Output matches original serial execution.
13. Sequential groups are excluded.
14. Unsafe nodes are excluded based on Final Issue 3 validation.
15. Benchmarks prove speedup on wide models.
16. Serial fallback remains available.
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
feature flag disabled fallback
missing preload rejected in parallel mode
sequential group exclusion
overlapping output columns excluded
unsafe node classification respected
```

---

## Out of Scope

```text
Parallelizing sequential groups
Changing calculation semantics
Changing output format
Changing Python DAG manager behavior
Custom work-stealing scheduler
Parallelizing unresolved dependency nodes
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
```

---

## Labels

```text
omni-calc
rust
rayon
parallel-execution
ready-layer-execution
ready-node-execution
scheduler
performance
preloaded-data
no-python-callbacks
feature-flag
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
  -> should happen before Final Issue 3, 4, 5, 6, and 7

Final Issue 3
  -> required before Final Issue 7
  -> useful before Final Issue 4, 5, and 6

Final Issue 4
  -> should happen before Final Issue 7 for actuals/sequential-heavy models

Final Issue 5
  -> should happen before Final Issue 7 for cross-object-heavy models

Final Issue 6
  -> can happen before Final Issue 7
  -> safer after Final Issue 2 and Final Issue 3

Final Issue 7
  -> final major implementation step
```

---

# Final Recommended Order

```text
1. Final Issue 1 — Execution Profiling, Benchmarks, and Performance Regression Baseline

2. Final Issue 2 — Data Structures, Shared Column Storage, Clone Reduction, Interned IDs, FormulaEvaluator Context, and Typed IDs

3. Final Issue 3 — Scheduler Foundation, Worker-Safe PreloadedMetadata, Resolver Refresh, Batched Merge, Parallel-Safety Validation, and Formula Dependency Graph Build

4. Final Issue 4 — Actuals, Forecast Indices, Entity Key Reuse, and Row Metadata

5. Final Issue 5 — Cross-Object Join Path Creation, Lookup Aggregation, and Join-Key Representation

6. Final Issue 6 — Pre/Post Processing Parallelism

7. Final Issue 7 — Configurable Rayon-Based Ready-Layer / Ready-Node Parallel Execution
```

---

# Final Notes

This corrected merged roadmap keeps all 21 source issues covered while reducing implementation tracking to 7 final Jira issues.

Most important corrections:

```text
Source Issue 13 is not benchmarking.
Source Issue 13 belongs in Final Issue 2 as interned IDs.

Source Issue 14 is not debug cleanup.
Source Issue 14 belongs in Final Issue 3 as dependency-aware resolver snapshot refresh.

Source Issue 16 is the benchmark/profiling issue.
Source Issue 16 belongs in Final Issue 1.

Source Issue 18 is not observability.
Source Issue 18 belongs in Final Issue 3 as formula dependency parsing and graph build.

Source Issue 15 is not standalone.
Source Issue 15 belongs in Final Issue 3 as batched NodeOutput merge.

Source Issue 17 is not standalone.
Source Issue 17 belongs in Final Issue 3 as pre-execution parallel-safety validation.

Source Issue 12 is not actuals/dimension metadata.
Source Issue 12 belongs in Final Issue 7 as Rayon ready-layer execution.
```

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
add dependency-aware resolver refresh
    ↓
batch NodeOutput merge
    ↓
validate parallel-safety before execution
    ↓
build formula dependency graph once
    ↓
build single-threaded Kahn scheduler
    ↓
prove serial parity
    ↓
optimize actuals / row metadata / joins / pre-post phases
    ↓
enable controlled Rayon ready-layer parallelism
```

This creates a safer and faster Omni-Calc engine without introducing race conditions, Python callback bottlenecks, nondeterministic output ordering, or global lock bottlenecks.
