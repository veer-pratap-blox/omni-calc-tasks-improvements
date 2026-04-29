# Code-Verified Final Omni-Calc Issue Reconciliation

Current branch reviewed:

```text
BLOX-2104-improve-preload-snapshot-py-rust-pass
```

Authoritative source:

```text
current branch code only
```

Prior documents reviewed:

```text
traction-react/29-april-omni-calc-parallelisation.md
traction-react/29-april-omni-calc-parallelisation-FINAL.md
traction-react/29-april-omni-calc-parallelisation-FINAL-MERGED.md
traction-react/29-april-omni-calc-parallelisation-FINAL-MERGED1.MD
```

Missing requested document:

```text
29-april-omni-calc-CACHE-improvements.md
```

That cache document was not present in this checkout. Cache-related findings below are therefore verified from code, not from that missing markdown file.

---

# 1. Code Reanalysis Findings

## Finding: Executor still runs calc steps and nodes serially

- Area/module: `modelAPI/omni-calc/src/engine/exec/executor.rs`, `modelAPI/omni-calc/src/engine/planner/plan.rs`, `modelAPI/omni-calc/src/engine/runtime.rs`
- Problem: `runtime::execute` wraps `CalcPlan` in a thin `Plan`, then `executor::execute` loops through `plan.request.calc_steps` in order. `process_input_step`, `process_calculation_step`, and `process_sequential_step` each process nodes through serial loops.
- Why it matters: The current branch does not provide node-level parallel execution. It only creates prerequisites such as Rust-owned preload data and releases the Python GIL around Rust execution.
- Suggested scope boundary: Build a Rust execution graph and single-thread Kahn scheduler before adding Rayon ready-node execution.
- Related existing docs/files: Initial tickets 2, 3, 4, 5, 6, 13, 15; FINAL issues 1, 2, 12, 17, 18; MERGED/MERGED1 final scheduler and Rayon issues.

## Finding: Execution state is globally mutable during node execution

- Area/module: `modelAPI/omni-calc/src/engine/exec/context.rs`, `modelAPI/omni-calc/src/engine/exec/executor.rs`
- Problem: `ExecutionContext` owns mutable `calc_object_states`, `resolver`, warning vectors, counters, and property caches. Node execution directly mutates all of these.
- Why it matters: Safe ready-node execution needs read-only inputs and bounded output merge semantics. A global `Arc<Mutex<ExecutionContext>>` would serialize expensive work and hide dependency mistakes.
- Suggested scope boundary: Introduce immutable `ExecutionSnapshot`, per-node `NodeOutput`, and a central merge path before enabling concurrency.
- Related existing docs/files: Initial tickets 1, 14; FINAL issues 1, 15; MERGED/MERGED1 broad scheduler foundation.

## Finding: Resolver refresh rebuilds full RecordBatches after many node updates

- Area/module: `modelAPI/omni-calc/src/engine/exec/context.rs`, `modelAPI/omni-calc/src/engine/exec/executor.rs`, `modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`
- Problem: `ctx.update_resolver(block_key)` calls `build_record_batch(state)` and replaces resolver block data. The executor calls this after input nodes, calculation nodes, sequential groups, and some connected-dimension additions.
- Why it matters: This is costly in serial execution and becomes a major synchronization boundary for parallel execution.
- Suggested scope boundary: Make resolver updates dependency-aware and batch them at layer or block boundaries where downstream references actually require refresh.
- Related existing docs/files: Initial ticket 7; FINAL issue 14; MERGED/MERGED1 scheduler foundation.

## Finding: `PreloadedMetadata` is only a partial Rust-owned metadata snapshot

- Area/module: `modelAPI/omni-calc/src/engine/exec/preload.rs`, `modelAPI/services/model_metadata_cache_v4.py`, `modelAPI/omni-calc/src/engine/exec/get_source_data/dim_loader.rs`
- Problem: Preload derives needs from `property_specs`, fetches dimension items and item property rows, then stores owned maps. It still reads the Python cache private `_item_properties_cache` as a fallback and legacy PyO3 metadata loaders remain in the execution module.
- Why it matters: Ready-node workers should never depend on Python objects, private cache internals, or GIL access. The current branch is close, but the contract is not fully normalized.
- Suggested scope boundary: Define a public bulk preload API and a complete worker-safe snapshot contract. Remove or quarantine unused PyO3 loaders from the executor path.
- Related existing docs/files: Initial ticket 12; FINAL issues 8, 10; cache-improvement topic.

## Finding: Property caches store maps, not target-aligned columns

- Area/module: `modelAPI/omni-calc/src/engine/exec/context.rs`, `modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs`, `modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs`
- Problem: `string_property_map_cache` and `numeric_property_map_cache` cache raw item-to-value maps. Each use still joins those maps to target row structure with `join_string_property_to_rows` or `join_property_map_to_rows`.
- Why it matters: Repeated property references on the same target shape redo row alignment work. Parallel execution would duplicate that work across workers unless aligned columns are cached as immutable outputs.
- Suggested scope boundary: Add cache keys that include property identity plus target row-shape identity, then cache aligned vectors separately from raw property maps.
- Related existing docs/files: Cache-improvement topic; FINAL issues 8, 10; MERGED/MERGED1 data/cache foundation.

## Finding: Column storage uses linear Vec scans and loose append semantics

- Area/module: `modelAPI/omni-calc/src/engine/exec/state.rs`, `modelAPI/omni-calc/src/engine/exec/context.rs`, `modelAPI/omni-calc/src/engine/exec/executor.rs`
- Problem: `CalcObjectState` stores columns as `Vec<(String, Vec<_>)>`. Lookups are linear scans, writes often push without a central replace policy, and `build_record_batch` only detects duplicate names late.
- Why it matters: This is a performance bottleneck and a correctness risk for merge-heavy execution. It also makes deterministic parallel merge semantics harder.
- Suggested scope boundary: Replace the Vec-backed store with an indexed ordered column store and explicit add/replace rules.
- Related existing docs/files: FINAL issue 7; FINAL issue 11; MERGED/MERGED1 data structure issue.

## Finding: Execution clones large vectors repeatedly

- Area/module: `modelAPI/omni-calc/src/engine/exec/executor.rs`, `modelAPI/omni-calc/src/engine/exec/context.rs`, `modelAPI/omni-calc/src/engine/exec/steps/calculation.rs`, `modelAPI/omni-calc/src/engine/exec/formula_eval.rs`
- Problem: The executor clones dimension, connected-dimension, number, string, and time vectors while building evaluation contexts and RecordBatches. `FormulaEvaluator::with_raw_properties` clones the full `EvalContext`.
- Why it matters: These clones can dominate runtime and memory. They also make Rayon execution memory-bound before CPU-bound.
- Suggested scope boundary: Introduce shared immutable column references and evaluator column views without changing scheduler behavior.
- Related existing docs/files: FINAL issues 8, 11, 19; MERGED/MERGED1 data structure issue.

## Finding: Formula parsing and dependency extraction happen late and repeatedly

- Area/module: `modelAPI/omni-calc/src/engine/exec/formula_eval.rs`, `modelAPI/omni-calc/src/engine/exec/executor.rs`, `modelAPI/omni-calc/src/engine/expr/parser`
- Problem: `FormulaEvaluator::evaluate` calls `parse_formula(formula)` per evaluation. Cross-object and dimension-property references are extracted from formula strings during node processing.
- Why it matters: Scheduler construction, parallel-safety validation, and execution all need the same dependency metadata. Re-parsing and string scanning at execution time adds avoidable work.
- Suggested scope boundary: Cache parsed AST and dependency metadata during planning/pre-execution, then reuse it for graph build and evaluation.
- Related existing docs/files: FINAL issue 18; MERGED/MERGED1 scheduler foundation.

## Finding: Actuals and forecast row metadata are rebuilt per indicator path

- Area/module: `modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs`, `modelAPI/omni-calc/src/engine/exec/steps/calculation.rs`
- Problem: Input and calculation handlers both rebuild dimension maps, forecast checks, actual-data maps, dimension keys, and entity keys using row-wise string construction.
- Why it matters: The work repeats across indicators that share the same block row structure. It is independent from scheduler work and can be optimized safely first.
- Suggested scope boundary: Precompute per-block row metadata, forecast masks, dimension keys, and entity keys.
- Related existing docs/files: FINAL issue 20; MERGED/MERGED1 actuals issue.

## Finding: Cross-object joins rebuild string join paths and lookup maps per reference

- Area/module: `modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`, `modelAPI/omni-calc/src/engine/exec/node_alignment/join_path.rs`, `modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs`
- Problem: Resolver calls build source paths, lookup maps, target paths, and aligned vectors for each cross-object reference. Join keys are concatenated strings.
- Why it matters: This is a high-cost path for cross-block formulas. It should be optimized before or alongside parallel work so workers do not duplicate expensive setup.
- Suggested scope boundary: Cache reusable join plans and typed join keys first; evaluate deterministic parallel aggregation as a separate follow-up.
- Related existing docs/files: Initial tickets 10, 11; FINAL issues 5, 6, 9, 13; MERGED/MERGED1 cross-object issue.

## Finding: Hot-path debug/warn/println output remains in execution paths

- Area/module: `modelAPI/omni-calc/src/python.rs`, `modelAPI/omni-calc/src/engine/exec/executor.rs`, `modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs`, `modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`
- Problem: The current branch logs many per-step, per-node, per-join, and per-input diagnostics at `warn!`, plus stdout timing with `println!`.
- Why it matters: Logging can distort benchmark results and slow large calculations. It should be cleaned up before performance baselines are accepted.
- Suggested scope boundary: Gate diagnostics behind targeted debug flags or `trace/debug`, and remove unconditional stdout printing.
- Related existing docs/files: MERGED final issue 1 combined this with benchmarks, but it should be a separate Jira issue.

## Finding: Safe pre/post execution parallelism is possible but should be split

- Area/module: `modelAPI/omni-calc/src/engine/exec/context.rs`, `modelAPI/omni-calc/src/engine/exec/executor.rs`
- Problem: Final block `RecordBatch` materialization is serial. Connected dimension preload also loops serially across blocks and mutates block state.
- Why it matters: These are independent from node scheduling but are not the same problem. Combining them hides different data ownership and correctness requirements.
- Suggested scope boundary: Split final `RecordBatch` materialization and connected dimension preload into separate tickets.
- Related existing docs/files: Initial tickets 8, 9; FINAL issues 3, 4; MERGED/MERGED1 final issue 6.

## Finding: Rayon exists, but not for ready-node scheduling

- Area/module: `modelAPI/omni-calc/Cargo.toml`, `modelAPI/omni-calc/src/engine/exec/function_impl`, `modelAPI/omni-calc/src/engine/exec/executor.rs`
- Problem: Rayon is already used in some function-level time helpers, but the executor does not use Rayon for independent input, property, or calculation nodes.
- Why it matters: A ready-node Rayon implementation should be a final-stage ticket after graph, snapshot, merge, validation, and benchmark foundations exist.
- Suggested scope boundary: Add configurable ready-node execution behind a feature flag or engine config after the single-thread scheduler is correct.
- Related existing docs/files: Initial tickets 4, 5, 6, 13; FINAL issues 2, 12; MERGED/MERGED1 Rayon issue.

---

# 2. Existing Document Audit

## Missing Cache Document

| Original file | Original issue/title | Status | Reason | Correct target file | Notes |
|---|---|---|---|---|---|
| `29-april-omni-calc-CACHE-improvements.md` | Entire file | Remove from direct audit | File is not present in this checkout. Cache issues are still code-verified from `preload.rs`, `model_metadata_cache_v4.py`, and property map caches. | New final merged file | Cache issues should be represented as focused final issues, not inferred from a missing document. |

## `29-april-omni-calc-parallelisation.md`

| Original issue/title | Status | Reason | Correct target file | Notes on what changed |
|---|---|---|---|---|
| Refactor Omni-Calc executor around immutable execution snapshots and per-node outputs | Keep / Update | Code confirms mutable `ExecutionContext`; scope is valid but should not include graph build or Rayon execution. | New final merged file | Split into `ExecutionSnapshot` / `NodeOutput` merge issue. |
| Build explicit Rust-side execution graph from CalcPlan for Kahn-style scheduling | Keep / Update | Code has only thin `Plan` wrapper and serial `calc_steps`. | New final merged file | Keep as graph plus single-thread scheduler foundation. |
| Implement single-threaded Kahn scheduler before enabling parallel execution | Merge | Same root as execution graph; not useful as a separate Jira ticket. | New final merged file | Merged into explicit graph and single-thread Kahn issue. |
| Parallelize independent input indicator execution within ready graph layers | Move / Merge | Valid future behavior, but depends on graph/snapshot/validation. | New final merged file | Merged into final Rayon ready-node issue. |
| Parallelize preloaded property map population and property node execution | Split | Mixes cache/preload improvement with future ready-node execution. | New final merged file | Split into preload snapshot contract, aligned property column cache, and Rayon ready-node issue. |
| Parallelize independent calculation nodes within graph ready layers | Move / Merge | Valid only after scheduler and merge foundation. | New final merged file | Merged into final Rayon ready-node issue. |
| Reduce resolver materialization frequency and make resolver snapshots read-only | Keep / Update | Code confirms frequent full `RecordBatch` rebuilds. | New final merged file | Rewritten as dependency-aware resolver refresh. |
| Parallelize final block RecordBatch materialization | Keep | Code confirms serial final materialization. | New final merged file | Kept as separate issue. |
| Parallelize connected dimension preload across blocks | Keep / Update | Code confirms serial preload and state mutation. | New final merged file | Kept separate from final materialization. |
| Parallelize join-path creation and target alignment for large cross-object joins | Update / Split | Cross-object path cost is real, but parallelization should not be first step. | New final merged file | Rewritten as reusable join plan/key optimization. |
| Evaluate parallel aggregation for lookup map creation with deterministic reductions | Keep / Update | Code confirms serial aggregation, but determinism and benchmarks must be explicit. | New final merged file | Kept as separate follow-up issue. |
| Remove or isolate remaining Python callback paths from Rust execution hot path | Update | Current hot path uses preloaded snapshot; remaining issue is snapshot contract and legacy PyO3 paths. | New final merged file | Rewritten as normalized worker-safe preload contract. |
| Add configurable Rayon thread-pool strategy for Omni-Calc parallel execution | Keep / Move | Valid future execution issue, not current implementation. | New final merged file | Moved to final-stage ready-node Rayon issue. |
| Do not use a single global Arc<Mutex<ExecutionContext>> for parallel execution | Merge | This is an acceptance constraint for snapshot/merge design, not its own ticket. | New final merged file | Merged into `ExecutionSnapshot` / `NodeOutput` issue. |
| Do not parallelize sequential groups initially | Merge | This is a scheduler validation rule, not a standalone implementation ticket. | New final merged file | Merged into parallel-safety and sequential-barrier issue. |

## `29-april-omni-calc-parallelisation-FINAL.md`

| Original issue/title | Status | Reason | Correct target file | Notes on what changed |
|---|---|---|---|---|
| Issue 1: Refactor Rust Omni-Calc Executor Toward Safe Kahn-Style DAG Scheduling Foundation | Split | Too broad: mixes snapshot, graph, resolver, preload, sequential barriers, and callback concerns. | New final merged file | Split into graph/scheduler, snapshot/output, resolver refresh, preload contract, validation. |
| Issue 2: Add Configurable Rayon-Based Parallel Execution for Independent Ready Nodes | Keep / Move | Valid but should come after prerequisites. | New final merged file | Kept as final-stage issue. |
| Issue 3: Parallelize Final RecordBatch Materialization Across Blocks | Keep | Code-supported and independent. | New final merged file | Kept as narrow issue. |
| Issue 4: Parallelize Connected Dimension Preload Across Blocks | Keep / Update | Code-supported but requires safe ownership of preload outputs. | New final merged file | Kept separate from RecordBatch materialization. |
| Issue 5: Parallelize Join-Path Creation and Target Alignment for Large Cross-Object Joins | Update | Root cost is repeated string join path creation and lookup setup, not necessarily Rayon first. | New final merged file | Rewritten as reusable join plan and typed key issue. |
| Issue 6: Evaluate Parallel Lookup-Map Aggregation with Deterministic Reductions | Keep / Update | Code-supported but should be benchmark-gated. | New final merged file | Kept as separate follow-up. |
| Issue 7: Optimize CalcObjectState Column Storage and Lookup | Keep | Code-supported by Vec storage and linear lookup. | New final merged file | Kept as indexed column store issue. |
| Issue 8: Reduce Cloning by Reusing Preloaded Data and Shared Column References | Split | Mixes preload normalization with shared column storage. | New final merged file | Split into preload contract and shared column/view issue. |
| Issue 10: Expand and Normalize PreloadedMetadata for Worker-Safe Execution | Keep / Update | Code confirms partial snapshot and Python private dict fallback. | New final merged file | Kept as cache/preload contract issue. |
| Issue 11: Introduce Snapshot-Friendly Column Storage Using Shared Arc Data | Merge / Update | Overlaps with clone reduction and column store work. | New final merged file | Merged into shared immutable column/view issue. |
| Issue 12: Add Rayon-Based Ready-Layer Parallel Execution Behind a Feature Flag | Merge | Same root as Issue 2. | New final merged file | Merged into final Rayon ready-node issue. |
| Issue 13: Replace Hot-Path String Lookups with Interned IDs | Keep / Update | Code uses string node IDs, column IDs, and join paths heavily. | New final merged file | Kept as structured identifiers issue, scoped away from cross-object join-key cache. |
| Issue 14: Add Dependency-Aware Resolver Snapshot Refresh | Keep | Code-supported by repeated `ctx.update_resolver`. | New final merged file | Kept as narrow resolver issue. |
| Issue 15: Batch NodeOutput Merge Operations | Merge | Requires NodeOutput abstraction first. | New final merged file | Merged into snapshot/output/merge issue. |
| Issue 16: Add Execution Profiling and Benchmarks | Keep / Update | Valid and should be first. | New final merged file | Kept as profiling and regression baseline issue. |
| Issue 17: Add Pre-Execution Validation for Parallel-Safety | Keep / Update | Valid and distinct from scheduler implementation. | New final merged file | Kept as validation/sequential barrier issue. |
| Issue 18: Optimize Formula Dependency Parsing and Graph Build | Split / Update | Dependency extraction belongs with graph; AST caching is distinct. | New final merged file | Split into graph issue and parsed AST/dependency metadata issue. |
| Issue 19: Optimize FormulaEvaluator Context Cloning and Shared Column Access | Keep | Code-supported by `with_raw_properties` cloning `EvalContext`. | New final merged file | Kept as formula evaluator column-view issue. |
| Issue 20: Optimize Actuals Handling, Forecast Indices, and Entity Key Reuse | Keep | Code-supported by repeated actual/forecast/key construction. | New final merged file | Kept as actuals row metadata issue. |
| Issue 21: Introduce Structured Node and Column Identifiers | Merge / Update | Overlaps with Issue 13. | New final merged file | Merged into structured identifiers issue. |

## `29-april-omni-calc-parallelisation-FINAL-MERGED.md`

| Original issue/title | Status | Reason | Correct target file | Notes on what changed |
|---|---|---|---|---|
| Final Issue 1: Add Omni-Calc Performance Benchmark Baseline and Remove Debug Hot-Path Overhead | Split | Benchmarking and log cleanup are separate work streams. | New final merged file | Split into profiling baseline and hot-path logging cleanup. |
| Final Issue 2: Optimize Data Structures, Clone Reduction, Shared References, FormulaEvaluator Context, and Typed IDs Foundation | Split | Too broad for Jira. | New final merged file | Split into column store, shared columns, formula evaluator views, and structured IDs. |
| Final Issue 3: Refactor Rust Omni-Calc Executor Toward Safe Kahn-Style DAG Scheduling Foundation, Worker-Safe PreloadedMetadata, and Cached Formula Dependency Metadata | Split | Too broad and mixes cache, graph, validation, and parser work. | New final merged file | Split into graph/scheduler, snapshot/output, preload contract, AST/dependency cache, resolver refresh, validation. |
| Final Issue 4: Optimize Actuals, Forecast, Dimension Row Metadata, and Entity Key Reuse | Keep / Update | Code-supported and focused enough after wording cleanup. | New final merged file | Kept as actuals row metadata issue. |
| Final Issue 5: Optimize Cross-Object Join Path Creation, Lookup Aggregation, and Join-Key Representation | Split | Join path reuse and parallel aggregation should be separate. | New final merged file | Split into join plan/key optimization and deterministic parallel aggregation evaluation. |
| Final Issue 6: Parallelize Safe Pre/Post Execution Processing: RecordBatch Materialization and Connected Dimension Preload | Split | These are independent flows with different state ownership. | New final merged file | Split into two separate issues. |
| Final Issue 7: Standalone Source Issue 15 placeholder | Remove / Merge | Placeholder is stale; source issue is concrete NodeOutput merge work. | New final merged file | Merged into snapshot/output issue. |
| Final Issue 8: Standalone Source Issue 16 placeholder | Remove / Move | Placeholder is stale; source issue is profiling and benchmarks. | New final merged file | Moved to profiling issue. |
| Final Issue 9: Standalone Source Issue 17 placeholder | Remove / Move | Placeholder is stale; source issue is pre-execution validation. | New final merged file | Moved to validation issue. |
| Final Issue 10: Add Configurable Rayon-Based Parallel Execution for Independent Ready Nodes | Keep | Valid final-stage issue. | New final merged file | Kept after prerequisites. |

## `29-april-omni-calc-parallelisation-FINAL-MERGED1.MD`

| Original issue/title | Status | Reason | Correct target file | Notes on what changed |
|---|---|---|---|---|
| Final Issue 1: Add Omni-Calc Execution Profiling, Benchmarks, and Performance Regression Baseline | Keep | Code-supported and correctly scoped. | New final merged file | Kept. |
| Final Issue 2: Optimize Data Structures, Shared Column Storage, Clone Reduction, Interned IDs, FormulaEvaluator Context, and Typed IDs | Split | Too broad for Jira. | New final merged file | Split into column store, shared columns, formula evaluator views, structured IDs. |
| Final Issue 3: Refactor Executor Toward Kahn-Style Scheduler Foundation, Worker-Safe PreloadedMetadata, Dependency-Aware Resolver Refresh, Batched NodeOutput Merge, Parallel-Safety Validation, and Formula Dependency Graph Build | Split | Too broad and mixes unrelated concerns. | New final merged file | Split into graph, snapshot/output, preload, resolver, validation, formula dependency cache. |
| Final Issue 4: Optimize Actuals, Forecast Indices, Entity Key Reuse, and Row Metadata | Keep | Code-supported and focused enough. | New final merged file | Kept. |
| Final Issue 5: Optimize Cross-Object Join Path Creation, Lookup Aggregation, and Join-Key Representation | Split | Join key reuse and parallel aggregation are different implementation scopes. | New final merged file | Split into cross-object join plan/key optimization and aggregation parallelism evaluation. |
| Final Issue 6: Parallelize Safe Pre/Post Processing: Final RecordBatch Materialization and Connected Dimension Preload | Split | Combines unrelated independent tasks. | New final merged file | Split into RecordBatch materialization and connected dimension preload issues. |
| Final Issue 7: Add Configurable Rayon-Based Ready-Layer / Ready-Node Parallel Execution | Keep / Move | Valid final-stage issue only after prerequisites. | New final merged file | Kept last. |

---

# 3. Corrected Issue Mapping

## Corrected Distribution By File

| File | Correct role after audit | Issues that belong there |
|---|---|---|
| `29-april-omni-calc-CACHE-improvements.md` | Missing from checkout; cannot be updated directly. | Cache/preload topics that would belong there: normalized `PreloadedMetadata`, public bulk cache API, aligned property-column cache. |
| `29-april-omni-calc-parallelisation.md` | Raw source notes only. | Keep as historical ticket discovery. Do not treat as final Jira set. |
| `29-april-omni-calc-parallelisation-FINAL.md` | Expanded source issue catalog. | Useful as source of candidate issues, but too broad in Issue 1 and has duplicates with later issues. |
| `29-april-omni-calc-parallelisation-FINAL-MERGED.md` | Superseded merged draft. | Contains stale placeholders and some incorrect broad merges. |
| `29-april-omni-calc-parallelisation-FINAL-MERGED1.MD` | Latest merged draft, but too broad. | Useful for high-level grouping only; must be split for Jira. |
| `29-april-omni-calc-parallelisation-FINAL-CODE-VERIFIED.md` | Authoritative final output. | Contains only code-supported, focused final issues. |

## Final Source-To-Issue Mapping

| Final issue | Source docs mapped into it |
|---|---|
| Profiling and benchmark baseline | FINAL issue 16; MERGED1 final issue 1 |
| Hot-path logging cleanup | MERGED final issue 1 partial; code-only verification |
| Indexed column store | FINAL issue 7; MERGED/MERGED1 final issue 2 partial |
| Shared immutable columns and clone reduction | FINAL issues 8, 11; MERGED/MERGED1 final issue 2 partial |
| FormulaEvaluator column views | FINAL issue 19; MERGED/MERGED1 final issue 2 partial |
| Structured node/column identifiers | FINAL issues 13, 21; MERGED/MERGED1 final issue 2 partial |
| Worker-safe preload snapshot | Initial ticket 12; FINAL issue 10; cache topic |
| Aligned property-column cache | FINAL issue 8 partial; cache topic; code-only verification |
| Rust execution graph and single-thread Kahn scheduler | Initial tickets 2, 3; FINAL issue 1 partial; FINAL issue 18 partial |
| ExecutionSnapshot, NodeOutput, central merge | Initial tickets 1, 14; FINAL issues 1, 15 |
| Dependency-aware resolver refresh | Initial ticket 7; FINAL issue 14 |
| Parallel-safety validation and sequential barriers | Initial ticket 15; FINAL issue 17 |
| Parsed AST and dependency metadata cache | FINAL issue 18 |
| Actuals and forecast row metadata reuse | FINAL issue 20; MERGED/MERGED1 final issue 4 |
| Cross-object join plan and join-key reuse | Initial ticket 10; FINAL issues 5, 9; MERGED/MERGED1 final issue 5 partial |
| Deterministic parallel lookup aggregation evaluation | Initial ticket 11; FINAL issue 6; MERGED/MERGED1 final issue 5 partial |
| Final RecordBatch materialization parallelism | Initial ticket 8; FINAL issue 3; MERGED/MERGED1 final issue 6 partial |
| Connected dimension preload parallelism | Initial ticket 9; FINAL issue 4; MERGED/MERGED1 final issue 6 partial |
| Configurable Rayon ready-node execution | Initial tickets 4, 5, 6, 13; FINAL issues 2, 12; MERGED/MERGED1 Rayon issues |

---

# 4. Final Merged Markdown Document

## Issue: Add Omni-Calc Execution Profiling, Benchmarks, and Regression Baseline

### Summary

Add a repeatable benchmark and profiling baseline for current omni-calc execution before making scheduler, caching, or data-structure changes.

The current code has many plausible bottlenecks, including resolver rebuilds, column cloning, formula parsing, join path creation, property alignment, and actuals row-key construction. Without a baseline, later tickets cannot prove improvement or detect regressions.

### Problem

The branch has no focused benchmark harness for executor phases such as preload, input steps, calculation steps, sequential steps, cross-object resolver work, property alignment, final `RecordBatch` creation, and Python boundary overhead.

### Why this should be a separate Jira issue

This is measurement infrastructure. It should not be mixed with implementation changes, otherwise performance results cannot be trusted.

### Scope

- In scope: add repeatable benchmarks for representative model sizes and formulas.
- In scope: add phase-level timings for preload, execution, resolver refresh, joins, formula evaluation, actuals handling, and final output building.
- In scope: define regression thresholds and reporting format.
- Not in scope: changing scheduler behavior or optimizing code paths.

### Affected areas

- `modelAPI/omni-calc/src/python.rs`
- `modelAPI/omni-calc/src/engine/exec/executor.rs`
- `modelAPI/omni-calc/src/engine/exec/context.rs`
- `modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`
- `modelAPI/omni-calc/src/engine/exec/node_alignment`

### Notes

Run this before and after each performance ticket. Existing unconditional debug output must be cleaned up before benchmark numbers are accepted.

## Issue: Remove or Gate Hot-Path Debug Logging and Stdout Timing

### Summary

Clean up execution-path logging so normal calculations do not emit per-node, per-step, per-input, or per-join diagnostics at warning level or stdout.

Current code includes multiple `warn!` diagnostics inside executor loops and resolver/join paths, plus `println!` runtime timing in the Python binding.

### Problem

Hot-path logs distort benchmark data and add overhead in large calculations. Some logs also contain hardcoded model/node references, which makes the executor noisy and harder to operate.

### Why this should be a separate Jira issue

This is operational/performance cleanup, not a scheduler or architecture refactor. It can be delivered independently and should happen before benchmark baselines.

### Scope

- In scope: move diagnostic logs to `debug` or `trace`, or gate them behind targeted debug helpers.
- In scope: remove unconditional `println!` timing from `python.rs` or replace it with structured trace instrumentation.
- In scope: remove hardcoded node/block debug branches from normal execution paths.
- Not in scope: changing calculation semantics.

### Affected areas

- `modelAPI/omni-calc/src/python.rs`
- `modelAPI/omni-calc/src/engine/exec/executor.rs`
- `modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs`
- `modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`

### Notes

This issue should be completed before using timings from the benchmark issue.

## Issue: Replace Vec-Based CalcObjectState Columns With Indexed Column Storage

### Summary

Replace `CalcObjectState` column vectors with a storage abstraction that provides fast lookup and explicit add/replace behavior.

The current state stores columns as `Vec<(String, Vec<f64>)>` and `Vec<(String, Vec<String>)>`. Reads use linear scans and writes often append directly.

### Problem

Linear lookup becomes expensive as column counts grow. Loose append semantics can create duplicate column names, and duplicate detection currently happens late during `RecordBatch` construction.

### Why this should be a separate Jira issue

This is a focused data-structure change. It supports both serial performance and future merge correctness but does not require scheduler changes.

### Scope

- In scope: introduce an ordered indexed column store for number, string, dimension, and connected-dimension columns.
- In scope: define explicit `insert_if_absent`, `replace`, and `upsert` semantics.
- In scope: update executor and handlers to use the new API.
- Not in scope: introducing shared `Arc` storage or Rayon execution.

### Affected areas

- `modelAPI/omni-calc/src/engine/exec/state.rs`
- `modelAPI/omni-calc/src/engine/exec/context.rs`
- `modelAPI/omni-calc/src/engine/exec/executor.rs`
- `modelAPI/omni-calc/src/engine/exec/steps`

### Notes

Sequential property replacement should be preserved explicitly because sequential steps intentionally replace prior property columns in some paths.

## Issue: Introduce Shared Immutable Column Storage and Column Views

### Summary

Reduce large vector cloning by representing stable input columns and calculated columns through shared immutable references where possible.

The executor currently clones dimension columns, connected dimension columns, number columns, time values, and evaluator contexts in several places.

### Problem

Repeated cloning increases memory pressure and can erase the benefit of later parallel execution. Future worker snapshots should pass cheap column views rather than copying row vectors.

### Why this should be a separate Jira issue

This is memory-layout and ownership work. It should build on the indexed column store but stay separate from scheduler implementation.

### Scope

- In scope: introduce shared column references such as `Arc<[T]>`, Arrow arrays, or equivalent immutable column views.
- In scope: update execution snapshots and evaluator inputs to borrow/share stable columns.
- In scope: avoid cloning dimension/time columns for every node where the row shape is unchanged.
- Not in scope: changing formula semantics or enabling ready-node Rayon execution.

### Affected areas

- `modelAPI/omni-calc/src/engine/exec/executor.rs`
- `modelAPI/omni-calc/src/engine/exec/context.rs`
- `modelAPI/omni-calc/src/engine/exec/formula_eval.rs`
- `modelAPI/omni-calc/src/engine/exec/steps/calculation.rs`

### Notes

This should be benchmarked independently because shared storage can improve serial execution even before parallelism.

## Issue: Reduce FormulaEvaluator Context Cloning With Shared Column Views

### Summary

Optimize `FormulaEvaluator` so child evaluators and function argument evaluation do not clone the full `EvalContext`.

Current `with_raw_properties` clones the entire context to disable property filtering for nested function behavior.

### Problem

Formula evaluation can copy large column maps and string vectors when evaluating nested expressions. This is separate from parser caching and should be addressed in the evaluator design.

### Why this should be a separate Jira issue

The work is localized to formula evaluation internals. It should not be bundled with scheduler or global column-store changes beyond using any shared column-view API that exists.

### Scope

- In scope: replace full-context child clones with borrowed/shared views where possible.
- In scope: preserve property filtering behavior for raw nested function arguments.
- In scope: keep warning collection and actuals context behavior correct.
- Not in scope: changing the parser or dependency graph.

### Affected areas

- `modelAPI/omni-calc/src/engine/exec/formula_eval.rs`
- `modelAPI/omni-calc/src/engine/exec/steps/calculation.rs`
- `modelAPI/omni-calc/src/engine/exec/steps/sequential.rs`

### Notes

This issue can be implemented after or alongside shared immutable column storage, but it should remain a focused evaluator ticket.

## Issue: Introduce Structured Node and Column Identifiers

### Summary

Replace repeated hot-path string parsing and string comparison for node IDs, block IDs, dimension columns, indicator columns, and property columns with structured identifiers.

The current code frequently checks prefixes such as `ind`, `prop`, `block`, and `_`, then parses or compares strings during execution.

### Problem

String-heavy identifiers add overhead and make dependency extraction, column lookup, and merge semantics more fragile. They also make it easier to mix display names, formula names, and storage keys.

### Why this should be a separate Jira issue

This is a representation cleanup across executor data structures. It should be separate from cross-object join-key optimization, which has its own row-key and aggregation concerns.

### Scope

- In scope: introduce typed IDs for node, block, dimension, property, and column references.
- In scope: keep string compatibility at Python/formula boundaries.
- In scope: update hot-path lookups to use structured keys internally.
- Not in scope: changing external formula syntax.

### Affected areas

- `modelAPI/omni-calc/src/engine/integration/calc_plan.rs`
- `modelAPI/omni-calc/src/engine/exec/state.rs`
- `modelAPI/omni-calc/src/engine/exec/executor.rs`
- `modelAPI/omni-calc/src/engine/exec/get_source_data/parser.rs`
- `modelAPI/omni-calc/src/engine/exec/node_alignment`

### Notes

External payload compatibility should be preserved by converting incoming string IDs once at the boundary.

## Issue: Normalize PreloadedMetadata Into a Worker-Safe Rust-Owned Snapshot

### Summary

Complete the preload contract so worker execution depends only on Rust-owned metadata and public bulk cache methods.

The current branch preloads dimension items and property maps, but `preload.rs` still uses the Python cache private `_item_properties_cache` as a fallback, and legacy PyO3 metadata loaders remain in the execution module.

### Problem

Future Rayon workers must not touch Python objects, private Python cache attributes, or GIL-bound callbacks. The current implementation is close to that model but should be formalized and cleaned up.

### Why this should be a separate Jira issue

This is a cache/preload contract issue. It should not be mixed with scheduler, resolver, or formula parsing work.

### Scope

- In scope: define the full metadata required by execution before `py.allow_threads`.
- In scope: use public bulk APIs for required metadata rather than private Python attributes.
- In scope: remove, quarantine, or clearly mark legacy PyO3 loader paths that must not be used by executor workers.
- Not in scope: node-level Rayon execution.

### Affected areas

- `modelAPI/omni-calc/src/engine/exec/preload.rs`
- `modelAPI/services/model_metadata_cache_v4.py`
- `modelAPI/omni-calc/src/engine/exec/get_source_data/dim_loader.rs`
- `modelAPI/omni-calc/src/engine/exec/metadata.rs`
- `modelAPI/omni-calc/src/python.rs`

### Notes

The current `get_dimension_items_bulk` and `get_item_property_values_bulk` methods are a good starting point, but the snapshot contract should be explicit and tested.

## Issue: Cache Target-Aligned Property Columns Separately From Raw Property Maps

### Summary

Extend property caching so repeated property references can reuse columns already aligned to a target row shape.

Current caches store raw property maps by `(dimension_id, property_id, scenario_id)`, but every use still joins those maps to the target rows.

### Problem

Repeated property references on the same block or dimension shape rebuild the same aligned vectors. This wastes serial time and would multiply under parallel workers.

### Why this should be a separate Jira issue

This is cache behavior, not metadata preload and not scheduler behavior. It is narrow and can be benchmarked independently.

### Scope

- In scope: define row-shape identity for block/dimension targets.
- In scope: cache aligned numeric, string, and connected-dimension property columns by property identity plus target row-shape identity.
- In scope: preserve actuals filtering rules where they affect aligned numeric property values.
- Not in scope: changing property semantics or formula evaluation behavior.

### Affected areas

- `modelAPI/omni-calc/src/engine/exec/context.rs`
- `modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs`
- `modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs`
- `modelAPI/omni-calc/src/engine/exec/get_source_data/dim_loader.rs`

### Notes

Actuals-filtered and unfiltered aligned property columns should use distinct cache keys.

## Issue: Build an Explicit Rust Execution Graph and Single-Thread Kahn Scheduler

### Summary

Create a Rust-side execution graph from `CalcPlan` and execute it with a single-thread Kahn-style scheduler before introducing node-level parallelism.

The current `Plan` is a thin wrapper around `CalcPlan`, and the executor trusts Python-provided `calc_steps` ordering.

### Problem

Rust does not currently own dependency scheduling. That prevents precise ready-node execution, validation, and dependency-aware resolver refresh.

### Why this should be a separate Jira issue

This issue creates scheduling structure without changing concurrency. It is the correctness foundation for later Rayon execution.

### Scope

- In scope: build graph nodes for input, property, calculation, and sequential groups.
- In scope: derive dependencies from parsed formula metadata, `node_maps`, `variable_filters`, property specs, and current `calc_steps` ordering.
- In scope: implement single-thread Kahn execution and verify parity with current serial ordering.
- Not in scope: running ready nodes in parallel.

### Affected areas

- `modelAPI/omni-calc/src/engine/planner/plan.rs`
- `modelAPI/omni-calc/src/engine/runtime.rs`
- `modelAPI/omni-calc/src/engine/exec/executor.rs`
- `modelAPI/omni-calc/src/engine/integration/calc_plan.rs`

### Notes

Sequential groups should initially be graph barrier nodes, not internally parallelized.

## Issue: Introduce ExecutionSnapshot, NodeOutput, and Centralized Merge

### Summary

Refactor node execution so workers read immutable inputs and return explicit outputs that are merged centrally.

Today node execution directly mutates `ExecutionContext`, block state, resolver state, warnings, and counters.

### Problem

Direct mutation makes safe parallel execution difficult. A single global lock around `ExecutionContext` would be correct but would serialize most expensive work.

### Why this should be a separate Jira issue

This is the core state-boundary refactor. It should not include graph scheduling, Rayon execution, or unrelated data-structure optimization.

### Scope

- In scope: define immutable `ExecutionSnapshot` inputs for node execution.
- In scope: define `NodeOutput` for number columns, string columns, connected-dimension columns, warnings, counts, and resolver refresh requests.
- In scope: implement deterministic central merge behavior.
- Not in scope: parallel execution or scheduler graph construction.

### Affected areas

- `modelAPI/omni-calc/src/engine/exec/context.rs`
- `modelAPI/omni-calc/src/engine/exec/executor.rs`
- `modelAPI/omni-calc/src/engine/exec/state.rs`
- `modelAPI/omni-calc/src/engine/exec/steps`

### Notes

This issue should reject a global `Arc<Mutex<ExecutionContext>>` design for node execution.

## Issue: Make CrossObjectResolver Refresh Dependency-Aware

### Summary

Reduce resolver rebuild frequency by refreshing resolver snapshots only when downstream references need updated block data.

Current `ctx.update_resolver` builds a full `RecordBatch` from block state and stores it in the resolver. The executor calls this repeatedly after node-level changes.

### Problem

Full `RecordBatch` rebuilds after many nodes add unnecessary work and create synchronization barriers for future parallel execution.

### Why this should be a separate Jira issue

Resolver refresh has distinct correctness rules around cross-object references. It should not be buried inside the scheduler or NodeOutput ticket.

### Scope

- In scope: track which blocks/nodes need resolver visibility for downstream references.
- In scope: refresh resolver snapshots at safe dependency boundaries.
- In scope: avoid rebuilding unchanged columns.
- Not in scope: changing cross-object join semantics.

### Affected areas

- `modelAPI/omni-calc/src/engine/exec/context.rs`
- `modelAPI/omni-calc/src/engine/exec/executor.rs`
- `modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`

### Notes

This should integrate with the graph scheduler once the graph exists, but the refresh policy is a separate topic.

## Issue: Add Pre-Execution Parallel-Safety Validation and Sequential Barriers

### Summary

Add validation that classifies which graph nodes can run concurrently and which must remain serial barriers.

Sequential groups, prior-period functions, resolver refresh needs, duplicate output columns, and shared mutable state assumptions must be identified before ready-node parallelism is enabled.

### Problem

The current executor relies on Python-provided ordering and does not validate whether a set of nodes is safe to run together.

### Why this should be a separate Jira issue

Validation is a correctness gate. It should be separate from both scheduler construction and Rayon execution.

### Scope

- In scope: validate dependencies, output column conflicts, sequential-group barriers, property-cache safety, and resolver visibility requirements.
- In scope: fail closed or fall back to serial execution when validation cannot prove safety.
- In scope: produce actionable warnings/errors for invalid graph states.
- Not in scope: parallelizing sequential groups internally.

### Affected areas

- `modelAPI/omni-calc/src/engine/planner`
- `modelAPI/omni-calc/src/engine/exec/executor.rs`
- `modelAPI/omni-calc/src/engine/exec/steps/sequential.rs`
- `modelAPI/omni-calc/src/engine/integration/calc_plan.rs`

### Notes

This is required before any ready-layer Rayon execution is enabled by default.

## Issue: Cache Parsed Formula AST and Dependency Metadata

### Summary

Parse formulas and extract dependency metadata during planning/pre-execution, then reuse the result for graph build, validation, and evaluation.

Current execution parses formula text in `FormulaEvaluator::evaluate` and extracts references from formula strings during dependency resolution.

### Problem

Repeated parsing and string scanning adds execution overhead and duplicates the same dependency analysis needed by the scheduler.

### Why this should be a separate Jira issue

Parser/dependency caching is a focused compiler/planner concern. It supports scheduler and evaluator performance without changing execution semantics.

### Scope

- In scope: store parsed ASTs for indicator formulas.
- In scope: store extracted local dependencies, cross-object references, dimension property references, lookup references, and sequential-function markers.
- In scope: update evaluator to consume cached ASTs where possible.
- Not in scope: changing formula syntax or adding Rayon execution.

### Affected areas

- `modelAPI/omni-calc/src/engine/expr/parser`
- `modelAPI/omni-calc/src/engine/exec/formula_eval.rs`
- `modelAPI/omni-calc/src/engine/exec/executor.rs`
- `modelAPI/omni-calc/src/engine/planner`

### Notes

This issue should preserve current Python-parity behavior, including existing property filtering rules.

## Issue: Precompute Actuals, Forecast Masks, Row Keys, and Entity Keys

### Summary

Precompute reusable row metadata for each block so input and calculation handlers do not rebuild the same maps and strings per indicator.

The current input and calculation paths each build dimension maps, actual maps, dimension keys, entity keys, and forecast-period checks.

### Problem

Repeated row-key construction and forecast checks add overhead proportional to row count times indicator count. The work is deterministic for a block row shape and can be reused.

### Why this should be a separate Jira issue

This is a focused execution-data optimization independent from scheduler and cache preload work.

### Scope

- In scope: build per-block row metadata once per execution.
- In scope: cache forecast masks/indices for the model forecast start and time granularity.
- In scope: cache dimension row keys and non-time entity keys.
- In scope: reuse metadata in input handling and actuals-aware calculation paths.
- Not in scope: changing actuals semantics.

### Affected areas

- `modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs`
- `modelAPI/omni-calc/src/engine/exec/steps/calculation.rs`
- `modelAPI/omni-calc/src/engine/exec/formula_eval.rs`

### Notes

Existing Python-compatibility edge cases such as raw input forecast-start zeroing must be preserved.

## Issue: Cache Cross-Object Join Plans and Reusable Join Keys

### Summary

Optimize cross-object reference alignment by caching reusable join plans and replacing repeated string join-path construction where possible.

Current resolver execution builds source join paths, lookup maps, target join paths, and aligned vectors for each reference.

### Problem

Cross-object joins can be one of the largest costs in models with many block references. Rebuilding the same path/key structures per reference duplicates work.

### Why this should be a separate Jira issue

This issue is about join planning and key reuse. It should not include parallel aggregation, scheduler changes, or global typed-ID migration.

### Scope

- In scope: cache join path/key vectors by row-shape and join dimension set.
- In scope: reuse source/target alignment plans across references with the same mapping.
- In scope: evaluate typed or interned join keys instead of concatenated strings.
- Not in scope: enabling Rayon aggregation or changing aggregation semantics.

### Affected areas

- `modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`
- `modelAPI/omni-calc/src/engine/exec/node_alignment/join_path.rs`
- `modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs`
- `modelAPI/omni-calc/src/engine/exec/node_alignment/property_bridge.rs`

### Notes

This should include correctness tests for direct joins, property bridges, aggregation, broadcasting, and filter-aware references.

## Issue: Evaluate Deterministic Parallel Lookup Aggregation

### Summary

Evaluate whether large grouped lookup-map aggregations should use Rayon, while preserving deterministic results.

Current lookup-map creation is serial and supports aggregation modes such as `sum`, `mean`, `first`, and `last`.

### Problem

Naively parallelizing aggregation can change ordering behavior for `first` and `last`, and can introduce floating-point differences for `sum` and `mean`.

### Why this should be a separate Jira issue

This is a narrow algorithmic performance experiment with determinism requirements. It should not be mixed with join-key caching.

### Scope

- In scope: benchmark serial versus parallel aggregation for large grouped joins.
- In scope: define deterministic behavior for `sum`, `mean`, `first`, and `last`.
- In scope: add a threshold or config for enabling parallel aggregation if it wins.
- Not in scope: general ready-node scheduler parallelism.

### Affected areas

- `modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs`
- `modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`

### Notes

This issue should run after the benchmark baseline and join-key reuse work.

## Issue: Parallelize Final RecordBatch Materialization Across Blocks

### Summary

Build final block `RecordBatch` outputs across independent block states in parallel after execution is complete.

Current final output materialization loops serially through `ctx.calc_object_states` and builds one batch per block.

### Problem

Final materialization clones column vectors into Arrow arrays and is independent across blocks. Large models can spend meaningful time here after calculation work is done.

### Why this should be a separate Jira issue

This is post-processing parallelism with simple independence boundaries. It should not be combined with connected dimension preload or ready-node execution.

### Scope

- In scope: parallelize final block batch creation where block states are immutable.
- In scope: preserve result ordering or define deterministic output ordering if required by callers.
- In scope: benchmark large block counts and wide block outputs.
- Not in scope: parallel node calculation.

### Affected areas

- `modelAPI/omni-calc/src/engine/exec/context.rs`
- `modelAPI/omni-calc/src/engine/exec/executor.rs`

### Notes

This becomes easier after shared immutable column storage, but it can be scoped independently.

## Issue: Parallelize Connected Dimension Preload Across Blocks

### Summary

Optimize connected dimension preload so independent block-level connected-dimension columns can be prepared concurrently or precomputed outside the main mutable executor path.

Current connected dimension preload loops serially across blocks and dimensions, builds columns, and then mutates each block state.

### Problem

The work is block-local but currently interleaved with mutable `ExecutionContext` state and noisy logging. It can be made safer and faster by computing block outputs independently and merging them.

### Why this should be a separate Jira issue

Connected dimension preload is a distinct pre-execution transformation. It has different correctness rules from final `RecordBatch` materialization.

### Scope

- In scope: compute connected dimension columns per block independently.
- In scope: merge connected dimension outputs into block state deterministically.
- In scope: preserve direct-dimension override and duplicate-column behavior.
- Not in scope: cross-object resolver refresh or node-level parallel execution.

### Affected areas

- `modelAPI/omni-calc/src/engine/exec/executor.rs`
- `modelAPI/omni-calc/src/engine/exec/context.rs`
- `modelAPI/omni-calc/src/engine/integration/calc_plan.rs`

### Notes

This should use the same output/merge pattern as node execution once available.

## Issue: Add Configurable Rayon Ready-Node Execution

### Summary

After graph, snapshot, merge, validation, and benchmarks are in place, add configurable Rayon execution for independent ready nodes.

The current branch already uses Rayon in some function-level time helpers, but the executor does not run independent nodes concurrently.

### Problem

Ready-node execution cannot be safely parallelized until dependencies, snapshots, output merge, resolver refresh, cache safety, and sequential barriers are explicit.

### Why this should be a separate Jira issue

This is the actual concurrency feature and should be the final stage, not mixed with prerequisite refactors.

### Scope

- In scope: execute ready graph layers or ready-node queues with Rayon behind config/feature flag.
- In scope: apply to safe input, property, and calculation nodes.
- In scope: keep deterministic merge order and serial fallback.
- In scope: benchmark thread counts and model sizes.
- Not in scope: parallelizing sequential groups internally.

### Affected areas

- `modelAPI/omni-calc/Cargo.toml`
- `modelAPI/omni-calc/src/engine/exec/executor.rs`
- `modelAPI/omni-calc/src/engine/planner`
- `modelAPI/omni-calc/src/types.rs`

### Notes

This issue should depend on the graph scheduler, snapshot/output merge, preload snapshot, validation, and benchmark tickets.

---

# 5. Change Summary

## Corrected Mappings

- The old "core scheduler foundation" issue was split into graph/scheduler, snapshot/output merge, resolver refresh, preload snapshot, validation, and parsed formula metadata.
- The old "data structures and typed IDs" issue was split into column store, shared column storage, evaluator context views, and structured identifiers.
- The old "cross-object joins" issue was split into reusable join plans/keys and deterministic parallel aggregation evaluation.
- The old "safe pre/post parallelism" issue was split into final `RecordBatch` materialization and connected dimension preload.

## Newly Added Issues

- Remove or gate hot-path debug logging and stdout timing.
- Cache target-aligned property columns separately from raw property maps.

## Split Issues

- `FINAL-MERGED1` Final Issue 2 split into four focused data/evaluator/identifier tickets.
- `FINAL-MERGED1` Final Issue 3 split into six focused scheduler/cache/resolver/validation/parser tickets.
- `FINAL-MERGED1` Final Issue 5 split into join-key reuse and aggregation parallelism tickets.
- `FINAL-MERGED1` Final Issue 6 split into two independent pre/post processing tickets.

## Merged Issues

- Initial single-thread Kahn scheduler ticket merged with explicit Rust execution graph.
- Initial "do not use global mutex" ticket merged into `ExecutionSnapshot` / `NodeOutput`.
- Initial "do not parallelize sequential groups initially" ticket merged into validation/sequential-barrier issue.
- FINAL issues 2 and 12 merged into final configurable Rayon ready-node issue.
- FINAL issues 13 and 21 merged into structured node/column identifiers.

## Removed Or Marked Invalid/Outdated

- `29-april-omni-calc-CACHE-improvements.md` could not be audited because it is missing from this checkout.
- `FINAL-MERGED.md` placeholder issues 7, 8, and 9 are outdated because the source issues are concrete and mapped elsewhere.
- Any claim that the current branch already performs ready-node parallel execution is invalid. Current executor execution remains serial.
- Any issue that depends on lazy Python callbacks inside Rayon workers was rewritten because current execution uses a preloaded snapshot, but the snapshot contract still needs cleanup.

