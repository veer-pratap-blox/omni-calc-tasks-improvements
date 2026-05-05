# Omni-Calc Rust Parallelism Refactor: Jira Issue Set

## Branch / Code Reference

Branch:

```text
Blox-Dev / BLOX-2104-improve-preload-snapshot-py-rust-pass
```

GitHub:

```text
https://github.com/BloxSoftware/Blox-Dev/tree/BLOX-2104-improve-preload-snapshot-py-rust-pass
```

Primary Rust areas reviewed:

```text
modelAPI/omni-calc/src/engine/exec/executor.rs
modelAPI/omni-calc/src/engine/exec/context.rs
modelAPI/omni-calc/src/engine/exec/preload.rs
modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs
modelAPI/omni-calc/src/engine/exec/get_source_data/dim_loader.rs
modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs
modelAPI/omni-calc/src/engine/exec/steps/calculation.rs
modelAPI/omni-calc/src/engine/exec/steps/sequential.rs
modelAPI/omni-calc/src/engine/integration/calc_plan.rs
modelAPI/omni-calc/src/python.rs
modelAPI/omni-calc/Cargo.toml
```

---

# High-Level Summary

This Jira issue set is based on the current Omni-Calc Rust execution model in the branch:

```text
BLOX-2104-improve-preload-snapshot-py-rust-pass
```

The branch has already moved metadata access toward Rust-side preload using `PreloadedMetadata`, which reduces Python callback dependency during execution. This creates a path toward real Rust-side parallel execution in the future.

However, the current executor still processes Python-provided `calc_steps` sequentially and mutates a shared `ExecutionContext` during node execution.

Current model:

```text
Python DAG manager creates ordered calc_steps
        ↓
Rust receives CalcPlan
        ↓
Rust creates one mutable ExecutionContext
        ↓
Rust loops through calc_steps in order
        ↓
Each node directly mutates shared state
        ↓
Resolver is updated during node processing
        ↓
Final RecordBatches are built
```

Target architecture:

```text
Current serial executor
    ↓
Immutable execution snapshots
    ↓
Per-node NodeOutput
    ↓
Central merge phase
    ↓
Explicit Rust-side execution graph
    ↓
Single-threaded Kahn scheduler
    ↓
Safe foundation for future parallelism
    ↓
Parallel ready-node execution in later follow-up issues
```

---

# Issue Grouping

The original 15 analysis tickets are grouped into the following Jira issues:

## Issue 1: Core Foundation Refactor

Merged original tickets:

```text
Ticket 1  - Immutable snapshots + NodeOutput + merge phase
Ticket 2  - Rust-side execution graph
Ticket 3  - Single-threaded Kahn scheduler
Ticket 7  - Resolver materialization / read-only resolver snapshot
Ticket 12 - Remove/isolate Python callback paths from Rust hot path
Ticket 14 - Reject global Arc<Mutex<ExecutionContext>>
Ticket 15 - Do not parallelize sequential groups initially
```

## Issue 2: Future Node-Level Parallel Execution

Merged original tickets:

```text
Ticket 4  - Parallelize independent input indicators
Ticket 5  - Parallelize property map population/property nodes
Ticket 6  - Parallelize independent calculation nodes
Ticket 13 - Configurable Rayon/thread-pool strategy
```

## Independent Performance Issues

These are separate follow-up issues because they can be implemented independently after the core executor structure is safer:

```text
Ticket 8  - Parallelize final RecordBatch materialization
Ticket 9  - Parallelize connected dimension preload
Ticket 10 - Parallelize join-path creation/alignment
Ticket 11 - Parallel aggregation for lookup maps
```

Final issue list:

```text
Issue 1 - Refactor Rust Omni-Calc Executor Toward Safe Kahn-Style DAG Scheduling Foundation
Issue 2 - Add Configurable Rayon-Based Parallel Execution for Independent Ready Nodes
Issue 3 - Parallelize Final RecordBatch Materialization Across Blocks
Issue 4 - Parallelize Connected Dimension Preload Across Blocks
Issue 5 - Parallelize Join-Path Creation and Target Alignment for Large Cross-Object Joins
Issue 6 - Evaluate Parallel Lookup-Map Aggregation with Deterministic Reductions
```

---

# ISSUE 1

## Issue Type

Technical Task / Refactor

---

## Title

Refactor Rust Omni-Calc Executor Toward Safe Kahn-Style DAG Scheduling Foundation

---

## Original Tickets Merged Into This Issue

```text
Ticket 1  - Immutable snapshots + NodeOutput + merge phase
Ticket 2  - Rust-side execution graph
Ticket 3  - Single-threaded Kahn scheduler
Ticket 7  - Resolver materialization / read-only resolver snapshot
Ticket 12 - Remove/isolate Python callback paths from Rust hot path
Ticket 14 - Reject global Arc<Mutex<ExecutionContext>>
Ticket 15 - Do not parallelize sequential groups initially
```

---

## Summary

Refactor the Rust `omni-calc` executor so it is structurally ready for a future Kahn-style parallel DAG scheduler.

The current Rust executor processes Python-provided `calc_steps` sequentially and mutates a shared `ExecutionContext` during node execution. This is safe for the current serial model, but it prevents safe, deterministic, and meaningful parallel execution of independent ready nodes.

This issue should create the architecture required for future parallelism, but it should **not** implement actual Rayon/thread-pool parallel execution yet.

The goal is to move from:

```text
calc_steps[0] -> calc_steps[1] -> calc_steps[2] -> ...
```

to a single-threaded version of:

```text
ready queue
    ↓
execute ready node
    ↓
return NodeOutput
    ↓
merge output centrally
    ↓
update resolver at safe boundary
    ↓
release dependent nodes
```

This gives the executor a correct dependency-driven foundation before any parallel worker execution is introduced.

---

## Background / Context

The current execution flow is mainly in:

```text
modelAPI/omni-calc/src/engine/exec/executor.rs
```

Current simplified flow:

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

Important point:

```rust
process_input_step(&mut ctx, step)
process_calculation_step(&mut ctx, step)
process_sequential_step(&mut ctx, step)
```

Each step receives mutable access to the same `ExecutionContext`.

The executor comments also state that the `calc_steps` are already ordered by the Python DAG manager, so Rust simply processes them in order.

Today Rust is effectively doing:

```text
Python-built topological step order
        ↓
Rust sequentially executes each step
        ↓
Each node mutates shared executor state
```

Rust is **not yet** doing:

```text
Build Rust-side graph
        ↓
Track indegrees
        ↓
Find dependency-free ready nodes
        ↓
Execute ready nodes
        ↓
Merge outputs
        ↓
Release dependents
```

---

## Current Graph Representation

The graph-like request data comes from:

```text
modelAPI/omni-calc/src/engine/integration/calc_plan.rs
```

Relevant structures:

```rust
pub struct CalcPlan {
    pub scenario_id: i64,
    pub forecast_start_date: Option<String>,
    pub model_start_date: Option<String>,
    pub time_granularity: String,
    pub fy_start_month: String,
    pub blocks: HashMap<String, BlockSpec>,
    pub dimensions: HashMap<String, DimensionSpec>,
    pub calc_steps: Vec<CalcStep>,
    pub metadata: CalcMetadata,
    pub node_maps: Vec<PlannedNodeMap>,
    pub variable_filters: HashMap<String, VariableFilter>,
    pub property_specs: HashMap<String, PropertySpec>,
}
```

Current step structure:

```rust
pub struct CalcStep {
    pub calc_type: String,
    pub nodes: Vec<String>,
}
```

Rust receives ordered steps like:

```json
[
  { "calc_type": "input", "nodes": ["ind1", "ind2"] },
  { "calc_type": "calculation", "nodes": ["ind3"] },
  { "calc_type": "sequential", "nodes": ["ind4", "ind5"] }
]
```

But Rust does not currently maintain:

```rust
struct ExecutionGraph {
    nodes: HashMap<NodeId, ExecNode>,
    indegree: HashMap<NodeId, usize>,
    outgoing: HashMap<NodeId, Vec<NodeId>>,
}
```

Dependency ordering is mostly implicit in `calc_steps`.

Cross-object mapping is represented separately using:

```rust
PlannedNodeMap
VariableFilter
PropertySpec
```

---

## Current Shared Mutable State

The shared state is defined in:

```text
modelAPI/omni-calc/src/engine/exec/context.rs
```

Current `ExecutionContext` concept:

```rust
pub struct ExecutionContext<'a> {
    pub calc_object_states: HashMap<String, CalcObjectState>,
    pub resolver: CrossObjectResolver,
    pub plan: &'a Plan,
    pub preloaded_metadata: Option<&'a PreloadedMetadata>,
    pub string_property_map_cache: HashMap<(i64, i64, i64), StringPropertyMap>,
    pub numeric_property_map_cache: HashMap<(i64, i64, i64), (PropertyMap, Vec<String>)>,
    pub input_handler: InputStepHandler,
    pub calc_handler: CalculationStepHandler,
    pub seq_handler: SequentialStepHandler,
    pub nodes_calculated: usize,
    pub warnings: Vec<CalcWarning>,
}
```

The following shared mutable fields are important for parallelism:

```text
ctx.calc_object_states
ctx.resolver
ctx.warnings
ctx.nodes_calculated
ctx.string_property_map_cache
ctx.numeric_property_map_cache
```

Today this is fine because execution is serial.

In a parallel-ready design, node execution should not directly mutate these shared fields.

---

## Current State Mutation During Calculation

Current calculation processing lives in:

```text
modelAPI/omni-calc/src/engine/exec/executor.rs
```

`process_calculation_step` loops node-by-node:

```rust
for node_id in &step.nodes {
    ...
}
```

Inside each node, the code may:

```rust
ctx.warnings.push(warning);
```

```rust
state.number_columns.push((col_name, values));
```

```rust
state.string_columns.push((col_name, values));
```

```rust
state.connected_dim_columns.push((col_name, values));
```

```rust
ctx.nodes_calculated += step_res.count;
```

```rust
ctx.update_resolver(&block_key);
```

So each node currently does:

```text
dependency resolution
+ property collection
+ cross-object resolution
+ formula evaluation
+ state mutation
+ warning collection
+ resolver update
```

This mixing of calculation and mutation is the main blocker for safe parallelism.

---

## Why `Arc<Mutex<ExecutionContext>>` Is Not Acceptable

A naive approach would be:

```rust
let ctx = Arc::new(Mutex::new(ctx));

ready_nodes.into_par_iter().for_each(|node| {
    let mut ctx = ctx.lock().unwrap();
    process_node(&mut ctx, node);
});
```

This is not a good production design.

Reason:

```text
The whole executor state is locked for the entire node execution.
Only one worker can actually run node logic at a time.
Expensive formula evaluation happens while holding the lock.
Resolver updates happen while holding the lock.
State mutation happens while holding the lock.
This provides thread safety but not true parallel speedup.
```

Rejected production design:

```rust
Arc<Mutex<ExecutionContext>>
```

around the full node lifecycle.

Preferred design:

```text
Build immutable snapshot
        ↓
Execute node without global mutation
        ↓
Return NodeOutput
        ↓
Merge output in one short controlled mutation phase
```

---

## Proposed Architecture

### 1. Keep `ExecutionContext` Coordinator-Owned

`ExecutionContext` should remain the owner of mutable execution state, but only the coordinator/scheduler should mutate it.

Workers or node execution functions should not take:

```rust
&mut ExecutionContext
```

for full node processing.

Target:

```text
ExecutionContext = coordinator-owned mutable state
```

---

### 2. Add Immutable `ExecutionSnapshot`

Add a read-only snapshot containing all data required for a node to calculate safely.

Example concept:

```rust
#[derive(Clone)]
struct ExecutionSnapshot {
    block_key: String,
    block_spec: Arc<BlockSpec>,

    dim_columns: Arc<Vec<(String, Vec<String>)>>,
    number_columns: Arc<Vec<(String, Vec<f64>)>>,
    string_columns: Arc<Vec<(String, Vec<String>)>>,
    connected_dim_columns: Arc<Vec<(String, Vec<String>)>>,

    resolver_snapshot: Arc<CrossObjectResolverSnapshot>,
    preloaded_metadata: Arc<PreloadedMetadata>,
    property_cache_snapshot: Arc<PropertyCacheSnapshot>,
}
```

Purpose:

```text
ExecutionSnapshot = read-only node input
```

A node should be executable using only:

```rust
fn execute_node(snapshot: ExecutionSnapshot, node: ExecNode) -> NodeOutput
```

No global mutable context should be required during calculation.

---

### 3. Add `NodeOutput`

Instead of directly pushing results into `CalcObjectState`, node execution should return an output object.

Example:

```rust
struct NodeOutput {
    block_key: String,

    number_columns: Vec<(String, Vec<f64>)>,
    string_columns: Vec<(String, Vec<String>)>,
    connected_dim_columns: Vec<(String, Vec<String>)>,

    warnings: Vec<CalcWarning>,
    nodes_calculated: usize,

    should_update_resolver: bool,
}
```

This makes the node execution output explicit.

Worker-like flow:

```rust
let output = execute_node(snapshot, node);
```

Coordinator flow:

```rust
merge_node_output(&mut ctx, output);
```

---

### 4. Add Centralized Merge Phase

Only the merge phase should mutate `ExecutionContext`.

Example:

```rust
fn merge_node_output(ctx: &mut ExecutionContext, output: NodeOutput) {
    if let Some(state) = ctx.calc_object_states.get_mut(&output.block_key) {
        for col in output.number_columns {
            if !state.number_columns.iter().any(|(name, _)| name == &col.0) {
                state.number_columns.push(col);
            }
        }

        for col in output.string_columns {
            if !state.string_columns.iter().any(|(name, _)| name == &col.0) {
                state.string_columns.push(col);
            }
        }

        for col in output.connected_dim_columns {
            if !state.connected_dim_columns.iter().any(|(name, _)| name == &col.0) {
                state.connected_dim_columns.push(col);
            }
        }
    }

    ctx.warnings.extend(output.warnings);
    ctx.nodes_calculated += output.nodes_calculated;

    if output.should_update_resolver {
        ctx.update_resolver(&output.block_key);
    }
}
```

This creates a controlled mutation point.

Important rule:

```text
Do not lock/mutate during calculation.
Only mutate during merge.
```

---

## Rust-Side Execution Graph

### Proposed Types

Add:

```rust
struct ExecutionGraph {
    nodes: HashMap<String, ExecNode>,
    outgoing: HashMap<String, Vec<String>>,
    indegree: HashMap<String, usize>,
}
```

Add:

```rust
struct ExecNode {
    id: String,
    block_key: String,
    calc_type: ExecNodeType,
    deps: Vec<String>,
    outputs: Vec<String>,
    parallel_safe: bool,
}
```

Add:

```rust
enum ExecNodeType {
    InputIndicator,
    Property,
    Calculation,
    SequentialGroup,
}
```

---

## Graph Construction Inputs

Build the graph from:

```text
CalcPlan.calc_steps
CalcPlan.blocks
BlockSpec.indicators
IndicatorSpec.parsed_formula
CalcPlan.property_specs
CalcPlan.node_maps
CalcPlan.variable_filters
```

---

## Dependency Tracking Requirements

### 1. Intra-Block Indicator Dependencies

If formula for `ind200` references `ind100`, graph should contain:

```text
ind100 -> ind200
```

Meaning:

```text
ind200 cannot execute until ind100 is merged.
```

---

### 2. Cross-Object Dependencies

Relevant resolver file:

```text
modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs
```

`CrossObjectResolver` stores calculated source block data:

```rust
pub struct CrossObjectResolver {
    calculated_blocks: HashMap<String, BlockData>,
    node_maps: HashMap<NodeMapKey, PlannedNodeMap>,
    variable_filters: HashMap<String, VariableFilter>,
}
```

Current resolver expects source block to already exist:

```rust
let source_block = self.calculated_blocks.get(&source_block_key)
    .ok_or_else(|| Error::eval_error(...))?;
```

A formula reference like:

```text
block39951___ind259068
```

means the target node must wait until:

```text
source block b39951 has ind259068 calculated
source block is materialized into resolver
resolver has needed connected dimension columns for joins/filters
```

In graph form:

```text
b39951.ind259068 -> target_node
```

---

### 3. Planned NodeMap Dependencies

`PlannedNodeMap` includes:

```rust
source_block_key
target_block_key
variable_name
source_dims_that_map
additional_columns_for_join
merge_on
groupby
property_join_columns
aggregation_mode
```

The graph should preserve dependencies implied by:

```text
source_block_key
target_block_key
variable_name
property_join_columns
additional_columns_for_join
```

If property bridge columns are required for a cross-object join, the source block snapshot must include them before the dependent node runs.

---

### 4. Variable Filter Dependencies

`VariableFilter` includes:

```rust
variable_name
source_block_key
source_node_id
filters
```

Filtered cross-object references should be treated as dependencies on:

```text
source block
source node
filter dimension/property availability
connected dimension columns if linked_dimension_id is used
```

---

### 5. Property Dependencies

Relevant files:

```text
modelAPI/omni-calc/src/engine/exec/preload.rs
modelAPI/omni-calc/src/engine/exec/get_source_data/dim_loader.rs
modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs
```

This branch adds `PreloadedMetadata`:

```rust
pub struct PreloadedMetadata {
    pub dimension_items: HashMap<i64, Vec<DimensionItem>>,
    pub property_maps: HashMap<(i64, i64, i64), HashMap<i64, String>>,
}
```

Property loading has Rust snapshot-based functions:

```rust
load_string_property_map_from_snapshot(...)
load_property_map_from_snapshot(...)
```

But legacy PyO3 callback functions still exist:

```rust
load_property_map(py, metadata_cache, ...)
load_string_property_map(py, metadata_cache, ...)
```

Graph/scheduler work must ensure future worker execution paths use preloaded Rust data only.

---

### 6. Sequential Group Dependencies

Sequential steps must be atomic.

Current sequential handler:

```text
modelAPI/omni-calc/src/engine/exec/steps/sequential.rs
```

Sequential formulas include:

```text
rollfwd(...)
prior(...)
balance(...)
change(...)
lookup(...)
```

These use period-by-period evaluation with entity state/history.

Do not split a sequential step like:

```text
nodes = [ind1, ind2, ind3]
```

into independent graph nodes.

Represent it as:

```rust
ExecNode {
    id: "seq_group_<id>",
    block_key,
    calc_type: ExecNodeType::SequentialGroup,
    deps,
    outputs: vec!["ind1", "ind2", "ind3"],
    parallel_safe: false,
}
```

---

## Single-Threaded Kahn Scheduler

Before adding Rayon or worker threads, add a single-threaded Kahn scheduler.

Pseudo-code:

```rust
let mut ready = VecDeque::new();

for node in graph.nodes.values() {
    if graph.indegree[&node.id] == 0 {
        ready.push_back(node.id.clone());
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
            ready.push_back(dep.clone());
        }
    }
}
```

The first implementation must remain single-threaded.

Purpose:

```text
Validate graph correctness.
Validate dependency ordering.
Validate resolver materialization timing.
Validate output parity with current calc_steps execution.
Keep sequential groups atomic.
```

---

## Resolver Materialization / Snapshot Strategy

Current resolver update:

```rust
pub fn update_resolver(&mut self, block_key: &str) {
    if let Some(state) = self.calc_object_states.get(block_key) {
        if let Ok(batch) = build_record_batch(state) {
            self.resolver.add_block(block_key.to_string(), batch);
        }
    }
}
```

Current issue:

```text
Node 1 -> rebuild full RecordBatch
Node 2 -> rebuild full RecordBatch
Node 3 -> rebuild full RecordBatch
```

`build_record_batch` clones:

```rust
StringArray::from(values.clone())
Float64Array::from(values.clone())
```

This is expensive and makes resolver mutation tightly coupled to node execution.

### Proposed Direction

Node execution should not call:

```rust
ctx.update_resolver(...)
```

directly.

Instead:

```text
execute node
return NodeOutput
merge output
update resolver at dependency-safe boundary
release dependents
```

### Read-Only Resolver Snapshot Concept

Add a read-only snapshot concept:

```rust
struct CrossObjectResolverSnapshot {
    calculated_blocks: Arc<HashMap<String, BlockData>>,
    node_maps: Arc<HashMap<NodeMapKey, PlannedNodeMap>>,
    variable_filters: Arc<HashMap<String, VariableFilter>>,
}
```

Future workers should read from resolver snapshots.

Workers should not mutate:

```rust
ctx.resolver
```

---

## Python Callback Boundary / Preload Requirement

Python boundary:

```text
modelAPI/omni-calc/src/python.rs
```

The branch preloads metadata before Rust execution:

```rust
let preloaded =
    crate::engine::exec::preload::preload_metadata(py, metadata_cache.as_ref(), &plan.inner)
        .map_err(pyo3::exceptions::PyRuntimeError::new_err)?;

engine.set_preloaded_metadata(preloaded);

let result = py
    .allow_threads(|| runtime::execute(&mut engine, plan.inner.clone()))
    .map_err(|e| pyo3::exceptions::PyRuntimeError::new_err(e.to_string()))?;
```

This is good because Rust execution happens inside:

```rust
py.allow_threads(...)
```

Meaning:

```text
Rust execution can run without holding the Python GIL.
```

However, worker-ready execution must not call Python at all.

### Required Mode

Add a conceptual mode:

```rust
enum MetadataAccessMode {
    PreloadedOnly,
    PythonFallbackAllowed,
}
```

Future parallel/worker-ready execution should require:

```rust
MetadataAccessMode::PreloadedOnly
```

If required metadata is missing, fail early with a clear error.

Do not lazy-load metadata from Python inside node execution.

---

## Scope

This issue includes:

```text
1. Introduce ExecutionSnapshot or equivalent read-only node input.
2. Introduce NodeOutput or equivalent per-node result.
3. Centralize state mutation in merge_node_output or equivalent.
4. Build explicit Rust-side ExecutionGraph.
5. Track node dependencies, outgoing edges, and indegrees.
6. Represent sequential steps as atomic non-parallel-safe nodes.
7. Implement single-threaded Kahn scheduler behind a feature flag/config.
8. Keep existing serial calc_steps execution path as default until parity is proven.
9. Refactor resolver updates to be coordinator-owned.
10. Add/read-only resolver snapshot concept for future worker execution.
11. Ensure worker-style execution paths use PreloadedMetadata only.
12. Reject global Arc<Mutex<ExecutionContext>> around full node execution.
13. Reject parallelizing sequential groups in this foundation issue.
```

---

## Out of Scope

This issue should not implement real parallel execution yet.

Out of scope:

```text
Full parallel scheduler with worker threads
Rayon/thread-pool integration
Work-stealing optimization
Parallelizing independent input indicators
Parallelizing property map population/property nodes
Parallelizing independent calculation nodes
Parallelizing final RecordBatch materialization
Parallelizing connected dimension preload
Parallelizing join-path creation/alignment
Parallel aggregation for lookup maps
Parallelizing sequential groups
Changing calculation semantics
Changing Python DAG manager behavior
Changing output format
```

---

## Proposed Implementation Plan

### Phase 1: Snapshot + Output Model

Add:

```rust
ExecutionSnapshot
NodeOutput
merge_node_output(...)
```

Refactor internals so node execution can eventually return `NodeOutput`.

Initial implementation can still run serially.

---

### Phase 2: Centralize Mutation

Move these operations into merge/coordinator functions:

```rust
ctx.warnings.push(...)
ctx.nodes_calculated += ...
state.number_columns.push(...)
state.string_columns.push(...)
state.connected_dim_columns.push(...)
ctx.update_resolver(...)
```

---

### Phase 3: Add ExecutionGraph

Add:

```rust
ExecutionGraph
ExecNode
ExecNodeType
```

Graph should include:

```text
node id
block key
node type
dependencies
outputs
parallel_safe
outgoing dependents
indegree
```

---

### Phase 4: Graph Diagnostics

Add diagnostics:

```text
print/debug node dependencies
print/debug ready nodes
detect cycles
compare graph topological order with current calc_steps order
```

---

### Phase 5: Single-Threaded Kahn Execution

Add:

```rust
execute_with_kahn_scheduler(ctx, graph)
```

behind a feature flag/config.

Keep existing execution path until parity is proven.

---

### Phase 6: Resolver Boundary Cleanup

Make resolver updates coordinator-owned.

Workers or node execution functions should not directly mutate resolver.

---

### Phase 7: Preloaded-Only Validation

Audit execution paths to ensure future worker-ready paths do not use:

```rust
Python<'_>
PyObject
metadata_cache.call_method1(...)
metadata_cache.getattr(...)
```

---

## Acceptance Criteria

### Architecture

- `ExecutionSnapshot` or equivalent exists.
- `NodeOutput` or equivalent exists.
- Node execution can be separated from global state mutation.
- Shared state mutation is centralized.
- Full `ExecutionContext` is not locked for entire node execution.
- Existing serial path remains available.

### Graph

- Rust can build an explicit graph from `CalcPlan`.
- Graph includes input nodes, property nodes, calculation nodes, and sequential groups.
- Graph tracks dependencies, outgoing edges, and indegrees.
- Sequential groups are represented as atomic nodes.
- Cross-object references become explicit dependencies.
- Property dependencies are represented or validated.
- Cycle detection exists.

### Scheduler

- Single-threaded Kahn scheduler exists behind config/feature flag.
- Kahn scheduler produces same outputs as current serial calc_steps execution.
- Resolver updates happen at safe boundaries.
- Dependent nodes are released only after required outputs are merged.

### Resolver

- Resolver mutation is coordinator-owned.
- Node execution does not directly call resolver updates.
- Read-only resolver snapshot concept exists.
- Cross-object references still resolve correctly.

### Python / Preload

- Future worker-style execution path uses `PreloadedMetadata`.
- Missing preloaded metadata fails early with a clear diagnostic.
- Python callback loaders are not used in worker-ready paths.
- No PyO3/Python callbacks should occur inside the Rust Omni-Calc execution hot path after preload. PyO3 should remain only at the Python boundary/preload stage.

### Rejected Designs

- No production implementation uses one global `Arc<Mutex<ExecutionContext>>` around full node execution.
- Sequential groups are not parallelized.

---

## Testing Notes

Add or update tests for:

### Snapshot / Output / Merge

```text
numeric column merge
string column merge
connected dimension column merge
duplicate column prevention
warning merge behavior
node count merge behavior
resolver update trigger behavior
```

### Graph Construction

```text
input -> calculation dependency
multiple independent inputs
intra-block formula dependency
cross-block blockX___indY dependency
property dependency
filtered cross-object dependency
sequential group as atomic node
cycle detection
topological order parity with calc_steps
```

### Kahn Scheduler

```text
pure input plan
simple calculated indicator
calculated indicator depending on another indicator
cross-block reference
property join
filtered reference
sequential step
actuals / forecast-start behavior
serial executor vs Kahn executor parity
```

### Resolver

```text
source block available before dependent node
source indicator exists in resolver batch
connected dimension columns included in resolver batch
resolver update after merge
missing source block diagnostic
missing indicator column diagnostic
```

### Python / Preload

```text
complete preload success
missing dimension items
missing property map
preloaded-only mode rejects missing metadata
no Python callback required during worker-ready execution
```

---

## Risks / Edge Cases

```text
Hidden dependencies may exist that are currently only protected by Python calc_steps ordering.
Cross-object references require resolver snapshots at precise times.
Connected dimension columns may be needed in resolver batches for property-bridge joins.
Warning order may change if graph order differs from calc_steps.
Column order may matter for downstream consumers.
Duplicate columns must be handled deterministically.
Sequential groups must remain atomic.
Preloaded metadata may not cover every legacy path.
Removing lazy Python fallback may expose missing metadata gaps.
Single-threaded Kahn must prove parity before any Rayon parallelism is attempted.
```

---

## Priority

Highest

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
python-ffi
gil
parallelism-foundation
tech-debt
performance-foundation
```

---

## Components

```text
Omni-Calc
Rust Engine
Calculation Executor
Dependency Scheduler
Cross-Object Resolver
Metadata Preload
Python Boundary
```

---

# ISSUE 2

## Issue Type

Epic / Story

---

## Title

Add Configurable Rayon-Based Parallel Execution for Independent Ready Nodes

---

## Original Tickets Merged Into This Issue

```text
Ticket 4  - Parallelize independent input indicators
Ticket 5  - Parallelize property map population/property nodes
Ticket 6  - Parallelize independent calculation nodes
Ticket 13 - Configurable Rayon/thread-pool strategy
```

---

## Summary

After Issue 1 lands and the executor has a safe graph/snapshot/output/merge architecture, add actual parallel execution for independent ready nodes.

This issue should use the Rust-side Kahn scheduler to find dependency-free ready nodes and execute safe nodes concurrently using Rayon.

Parallelize only nodes that are proven independent and safe:

```text
input indicator nodes
property nodes using preloaded metadata
calculation nodes with complete dependencies
```

Do not parallelize sequential groups.

---

## Background / Context

The branch already includes:

```text
rayon = "1.10"
```

in:

```text
modelAPI/omni-calc/Cargo.toml
```

However, current Rust execution does not yet use Rayon for executor scheduling.

The branch also moves metadata toward Rust-side preload, which allows property-related execution to avoid Python callbacks when `PreloadedMetadata` is complete.

---

## Problem Statement

Even after Issue 1, execution will still be single-threaded.

The next opportunity is to run independent ready nodes concurrently.

Examples:

```text
multiple input indicators in the same ready layer
multiple property nodes that only read preloaded metadata
multiple calculation nodes whose dependencies are already merged
```

Current executor processes these sequentially.

---

## Scope

This issue includes:

```text
1. Add configurable Rayon/thread-pool strategy.
2. Parallelize independent input indicator nodes.
3. Parallelize property map population/property node execution where metadata is preloaded.
4. Parallelize independent calculation nodes.
5. Preserve deterministic merge order.
6. Keep sequential groups excluded.
7. Keep serial fallback mode.
```

---

## Technical Analysis

### Input Nodes

Input loading is handled in:

```text
modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs
```

Input processing includes:

```text
parse data_values_json
detect input type
build DimensionMapper
build TimeUtils
load constant/raw/growth/constant_by_year values
apply actuals / forecast-start behavior
```

This is mostly local CPU work and does not require Python callbacks.

Parallelism level:

```text
independent calc execution
dependency graph layer execution
```

---

### Property Nodes

Property data is now available through:

```text
PreloadedMetadata
```

Snapshot-based functions:

```rust
load_string_property_map_from_snapshot(...)
load_property_map_from_snapshot(...)
```

Property maps should be prebuilt or read-only during node execution.

Parallelism level:

```text
cache population
preloaded metadata processing
independent property node execution
```

---

### Calculation Nodes

Formula evaluation is Rust-local via:

```text
modelAPI/omni-calc/src/engine/exec/steps/calculation.rs
modelAPI/omni-calc/src/engine/exec/formula_eval.rs
```

A calculation node can run in parallel if:

```text
all dependencies are already merged
required cross-object columns are available in the snapshot
required property columns are available from immutable property cache
it writes unique output columns
it is not a sequential group
it does not depend on another node in the same batch
```

Parallelism level:

```text
independent calc execution
dependency graph layer execution
batch formula evaluation
```

---

## Proposed Change

### Add Config

Add or extend engine config:

```rust
pub struct EngineConfig {
    pub enable_parallel_execution: bool,
    pub parallel_threads: Option<usize>,
    pub parallel_node_threshold: usize,
    pub parallel_row_threshold: usize,
}
```

### Add Parallel Execution Path

Conceptual flow:

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

### Deterministic Merge

Even if execution is parallel, merge must be deterministic.

Recommended:

```text
sort outputs by graph order / node order / original calc_steps order before merge
```

---

## Why Rayon Is Appropriate

Rayon is appropriate because this is CPU-bound Rust work:

```text
formula evaluation
input value generation
property map processing
vector calculations
```

Async is not appropriate for these execution paths because there is no IO wait inside worker node execution after preload.

Work-stealing should not be customized initially. Rayon’s default scheduler is sufficient for first implementation.

---

## Risks / Edge Cases

```text
Oversubscription if Python service handles many requests concurrently.
Nested Rayon usage could cause excessive parallelism.
Parallel output order must be deterministic.
Warnings must be stable or explicitly ordered.
Memory usage may increase when many large nodes run concurrently.
Hidden dependencies must be caught by graph construction.
Property cache must be immutable/read-only during worker execution.
Workers must not mutate ExecutionContext.
Workers must not mutate resolver.
Workers must not call Python.
```

---

## Dependencies

Requires Issue 1.

Specifically requires:

```text
ExecutionSnapshot
NodeOutput
central merge phase
ExecutionGraph
single-threaded Kahn scheduler parity
preloaded-only worker path
resolver snapshot safety
```

---

## Acceptance Criteria

- Parallel execution can be enabled/disabled by config.
- Serial fallback remains available.
- Rayon worker count can be configured or controlled.
- Independent input nodes can execute in parallel.
- Property nodes can execute in parallel using preloaded metadata only.
- Independent calculation nodes can execute in parallel.
- Workers do not mutate `ExecutionContext`.
- Workers do not mutate `CrossObjectResolver`.
- Workers do not call Python.
- Merge order is deterministic.
- Output matches serial Kahn execution.
- Sequential groups are excluded.

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
```

---

## Out of Scope

```text
Parallelizing sequential groups
Changing calculation semantics
Changing Python DAG manager behavior
Changing output format
Custom work-stealing scheduler
Parallel RecordBatch materialization
Parallel connected dimension preload
Parallel join-path alignment
Parallel lookup-map aggregation
```

---

## Priority

High

---

## Labels

```text
omni-calc
rust
rayon
parallel-execution
input-nodes
property-nodes
calculation-nodes
scheduler
performance
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

# ISSUE 3

## Issue Type

Performance Task

---

## Title

Parallelize Final RecordBatch Materialization Across Blocks

---

## Original Ticket

```text
Ticket 8 - Parallelize final RecordBatch materialization
```

---

## Summary

Parallelize final output `RecordBatch` construction across blocks after execution completes.

This is an independent performance optimization because final result materialization is naturally per-block and happens after executor mutation is complete.

---

## Background / Context

Final result building is in:

```text
modelAPI/omni-calc/src/engine/exec/context.rs
```

Current result building loops through `ctx.calc_object_states` and calls:

```rust
build_record_batch(state)
```

for each block.

`build_record_batch` clones columns into Arrow arrays:

```rust
StringArray::from(values.clone())
Float64Array::from(values.clone())
```

---

## Problem Statement

Final block materialization is sequential.

For models with many blocks or large block states, this can be CPU/memory-copy heavy.

Since final execution state should be read-only at this point, each block’s `RecordBatch` can be built independently.

---

## Proposed Change

Use Rayon to build batches per block:

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

Then sort and insert deterministically:

```rust
let mut batches = batches;
batches.sort_by(|a, b| a.0.cmp(&b.0));

for (block_key, batch) in batches {
    result.add_block(block_key, batch);
}
```

---

## Parallelism Opportunity

```text
snapshot/materialization work
per-block final output build
```

---

## Expected Impact

```text
Faster final result build for many-block models.
Low risk because state is read-only after execution.
No dependency graph changes required.
```

---

## Risks / Edge Cases

```text
Higher memory pressure because multiple Arrow arrays may be built at once.
Block output ordering must remain deterministic.
Errors must be handled consistently with current serial behavior.
```

---

## Dependencies

Can be done independently.

Recommended after Issue 1 for cleaner state ownership.

---

## Acceptance Criteria

- Final `RecordBatch` creation can run in parallel per block.
- Output matches serial implementation.
- Block ordering is deterministic.
- Errors are handled consistently.
- Serial fallback exists or can be configured.
- Tests cover multiple block outputs and schema parity.

---

## Testing Notes

Add tests for:

```text
multiple block outputs
numeric columns
string columns
connected dimension columns
duplicate column warning behavior
serial vs parallel result parity
large synthetic block materialization benchmark
```

---

## Out of Scope

```text
Changing RecordBatch schema
Changing output format
Parallelizing per-column materialization inside one block
Changing calculation execution
```

---

## Priority

Medium

---

## Labels

```text
omni-calc
rust
recordbatch
arrow
rayon
materialization
quick-win
performance
```

---

## Components

```text
Omni-Calc
Rust Engine
Result Materialization
Arrow Output
```

---

# ISSUE 4

## Issue Type

Performance Task / Tech Debt

---

## Title

Parallelize Connected Dimension Preload Across Blocks

---

## Original Ticket

```text
Ticket 9 - Parallelize connected dimension preload
```

---

## Summary

Refactor `preload_connected_dimensions` so connected dimension columns are computed per block in parallel and merged afterward.

---

## Background / Context

Connected dimension preload is currently in:

```text
modelAPI/omni-calc/src/engine/exec/executor.rs
```

Function:

```rust
fn preload_connected_dimensions(ctx: &mut ExecutionContext)
```

Current behavior:

```text
for each block
    for each block dimension
        inspect DimensionSpec.property_values
        build connected dimension columns
        mutate state.connected_dim_columns
```

This is currently sequential and mixes computation with state mutation.

---

## Problem Statement

Connected dimension preload is mostly independent per block.

However, current implementation directly reads and mutates `ExecutionContext`.

This prevents safe parallel execution and makes the logic harder to reason about.

---

## Proposed Change

Split into compute and merge phases.

Compute phase:

```rust
fn compute_connected_dims_for_block(
    snapshot: ConnectedDimPreloadSnapshot,
    block_key: String,
) -> BlockConnectedDimOutput
```

Output:

```rust
struct BlockConnectedDimOutput {
    block_key: String,
    connected_dim_columns: Vec<(String, Vec<String>)>,
    warnings: Vec<CalcWarning>,
}
```

Parallel compute:

```rust
let outputs: Vec<BlockConnectedDimOutput> = block_keys
    .par_iter()
    .map(|block_key| compute_connected_dims_for_block(snapshot.clone(), block_key.clone()))
    .collect();
```

Deterministic merge:

```rust
for output in stable_order(outputs) {
    merge_connected_dim_output(&mut ctx, output);
}
```

---

## Parallelism Opportunity

```text
preload work
per-block preprocessing
cache population
```

---

## Expected Impact

```text
Faster preload for models with many blocks/dimensions.
Cleaner compute/merge separation.
Less shared mutable state during preload.
```

---

## Risks / Edge Cases

```text
Duplicate connected dimension columns must still be skipped.
Merge order must be deterministic.
Existing warning/logging behavior may change.
Parallel logging may become noisy.
Must avoid reading mutable state while merging.
```

---

## Dependencies

Can be done independently.

Recommended after Issue 1 or with a local compute/merge split.

---

## Acceptance Criteria

- Connected dimension computation does not mutate `ExecutionContext`.
- Per-block connected dimension computation can run in parallel.
- Merge preserves current output behavior.
- Duplicate column handling remains correct.
- Serial and parallel preload outputs match.
- Logging remains usable.

---

## Testing Notes

Add tests for:

```text
block with no connected dimensions
block with connected dimension property values
multiple blocks with connected dimensions
duplicate connected dimension columns
missing dimension specs
empty property_values
serial vs parallel preload parity
```

---

## Out of Scope

```text
Changing connected dimension semantics
Changing Python payload format
Parallelizing formula execution
Parallelizing property map population
```

---

## Priority

Medium

---

## Labels

```text
omni-calc
rust
preload
connected-dimensions
rayon
performance
```

---

## Components

```text
Omni-Calc
Rust Engine
Preload
Connected Dimensions
```

---

# ISSUE 5

## Issue Type

Performance Task

---

## Title

Parallelize Join-Path Creation and Target Alignment for Large Cross-Object Joins

---

## Original Ticket

```text
Ticket 10 - Parallelize join-path creation/alignment
```

---

## Summary

Add threshold-based parallel implementations for join-path creation and target alignment in node alignment code.

---

## Background / Context

Cross-object alignment uses:

```text
modelAPI/omni-calc/src/engine/exec/node_alignment/join_path.rs
modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs
modelAPI/omni-calc/src/engine/exec/node_alignment/mod.rs
modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs
```

Current join path build:

```rust
pub fn build_all_join_paths(
    dim_columns: &[(String, Vec<String>)],
    dim_names: &[String],
) -> Vec<String> {
    (0..row_count)
        .map(|row_idx| build_join_path(dim_columns, dim_names, row_idx))
        .collect()
}
```

Current target alignment:

```rust
for path in target_join_paths {
    let value = lookup.get(path).copied().unwrap_or(default_value);
    result.push(value);
}
```

---

## Problem Statement

For large row counts, join path construction and target alignment can be CPU/memory intensive.

These operations are currently sequential.

---

## Proposed Change

Add parallel variants:

```rust
pub fn build_all_join_paths_parallel(...)
pub fn align_with_lookup_parallel(...)
```

Use threshold-based fallback:

```text
if row_count < threshold:
    use serial implementation
else:
    use parallel implementation
```

Example:

```rust
use rayon::prelude::*;

(0..row_count)
    .into_par_iter()
    .map(|row_idx| build_join_path(dim_columns, dim_names, row_idx))
    .collect()
```

For alignment:

```rust
target_join_paths
    .par_iter()
    .map(|path| lookup.get(path).copied().unwrap_or(default_value))
    .collect()
```

---

## Parallelism Opportunity

```text
cross-object alignment
row-level batch evaluation
read-only lookup usage
```

---

## Expected Impact

```text
Faster cross-block joins for large row counts.
Useful for models with heavy cross-object references.
No full scheduler parallelism required.
```

---

## Risks / Edge Cases

```text
String allocation is heavy and parallelization may increase memory pressure.
Small datasets may become slower due to Rayon overhead.
Output ordering must remain identical to serial implementation.
Debug counters and missing-count stats need reduction-safe logic.
HashMap must be read-only during parallel access.
```

---

## Dependencies

Can be implemented independently.

Recommended after Issue 1 and Issue 7-style resolver cleanup.

---

## Acceptance Criteria

- Parallel path is threshold-based.
- Serial fallback remains available.
- Output ordering matches serial implementation.
- Missing/default value behavior matches serial implementation.
- Tests prove serial/parallel parity.
- Benchmarks show improvement on large row counts.

---

## Testing Notes

Add tests for:

```text
simple join path creation
multi-dimension join path creation
missing target keys
target alignment with default values
large row synthetic benchmark
threshold fallback behavior
serial vs parallel parity
```

---

## Out of Scope

```text
Parallel lookup-map aggregation
Replacing string keys with encoded keys
Changing join semantics
Changing resolver behavior
```

---

## Priority

Medium

---

## Labels

```text
omni-calc
rust
node-alignment
join-path
cross-object
rayon
performance
```

---

## Components

```text
Omni-Calc
Rust Engine
Node Alignment
Cross-Object Resolver
```

---

# ISSUE 6

## Issue Type

Performance Research / Optimization

---

## Title

Evaluate Parallel Lookup-Map Aggregation with Deterministic Reductions

---

## Original Ticket

```text
Ticket 11 - Parallel aggregation for lookup maps
```

---

## Summary

Investigate and implement parallel lookup-map aggregation for large cross-object joins only if deterministic and measurably faster than the current sequential `HashMap` aggregation.

---

## Background / Context

Lookup map creation is in:

```text
modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs
```

Current aggregation concept:

```rust
for (path, &value) in join_paths.iter().zip(values.iter()) {
    *lookup.entry(path.clone()).or_insert(0.0) += value;
}
```

Supported aggregation modes include:

```text
sum
mean
first
last
```

---

## Problem Statement

For large source row counts, lookup map creation can be expensive.

However, naive parallelization using a shared `HashMap` would require locking and may be slower or nondeterministic.

---

## Proposed Change

Use per-worker local maps and deterministic reduction.

Conceptual approach:

```text
split rows into chunks
each worker builds local HashMap
merge local maps deterministically
```

Aggregation representation:

```rust
enum AggValue {
    Sum(f64),
    Mean { sum: f64, count: usize },
    First { index: usize, value: f64 },
    Last { index: usize, value: f64 },
}
```

Rules:

```text
sum  -> reduce by addition
mean -> reduce sum and count
first -> keep lowest original row index
last -> keep highest original row index
```

---

## Parallelism Opportunity

```text
aggregation/reduction
large cross-object joins
lookup-map construction
```

---

## Expected Impact

```text
Potential speedup for large grouped cross-object references.
Complements Issue 5.
May reduce cost of CrossObjectResolver alignment.
```

---

## Risks / Edge Cases

```text
Floating-point summation order may change tiny numeric results.
first/last must preserve original row order semantics.
Per-thread maps increase memory usage.
Parallel version may be slower for small/medium inputs.
Must benchmark before enabling by default.
```

---

## Dependencies

Recommended after Issue 5.

---

## Acceptance Criteria

- Parallel aggregation is deterministic.
- Modes match serial behavior:
  - sum
  - mean
  - first
  - last
- Threshold-based fallback exists.
- Floating-point tolerance is documented.
- Benchmarks justify enabling the parallel path.
- Serial implementation remains available.

---

## Testing Notes

Add tests for:

```text
sum aggregation parity
mean aggregation parity
first aggregation by row index
last aggregation by row index
duplicate join paths
empty input
large synthetic grouped input
deterministic output across repeated runs
serial vs parallel benchmark
```

---

## Out of Scope

```text
Changing aggregation semantics
Replacing join path string keys
Changing resolver behavior
Full scheduler parallelism
```

---

## Priority

Low / Medium

---

## Labels

```text
omni-calc
rust
lookup-map
aggregation
deterministic-reduction
rayon
performance-research
```

---

## Components

```text
Omni-Calc
Rust Engine
Node Alignment
Cross-Object Resolver
```

---

# Recommended Implementation Order

```text
1. Issue 1 - Core Kahn-style scheduler foundation
2. Issue 2 - Configurable Rayon-based parallel ready-node execution
3. Issue 3 - Parallel final RecordBatch materialization
4. Issue 4 - Parallel connected dimension preload
5. Issue 5 - Parallel join-path creation and target alignment
6. Issue 6 - Parallel lookup-map aggregation research
```

---

# Dependency Map

```text
Issue 1
  -> required before Issue 2

Issue 2
  -> depends on Issue 1

Issue 3
  -> can be done independently
  -> safer after Issue 1

Issue 4
  -> can be done independently with local compute/merge split
  -> cleaner after Issue 1

Issue 5
  -> can be done independently
  -> benefits from resolver cleanup in Issue 1

Issue 6
  -> should follow Issue 5
```

---

# Quick Wins vs Larger Refactors

## Larger Refactors

```text
Issue 1 - Core Kahn-style scheduler foundation
Issue 2 - Configurable Rayon-based parallel ready-node execution
```

Why:

```text
They change executor architecture and scheduling.
They require strong output parity tests.
They touch shared state, resolver behavior, and dependency ordering.
```

## Quick / Medium Wins

```text
Issue 3 - Parallel final RecordBatch materialization
Issue 4 - Parallel connected dimension preload
Issue 5 - Parallel join-path creation/alignment
```

Why:

```text
They are more localized.
They are naturally per-block or per-row.
They do not require full scheduler parallelism.
```

## Research / Benchmark First

```text
Issue 6 - Parallel lookup-map aggregation
```

Why:

```text
Correctness is more subtle because aggregation modes must remain deterministic.
Floating-point reduction order may change tiny numeric results.
Must benchmark before enabling by default.
```

---

# Final Notes

The branch’s preload changes create a real opportunity for Rust-side parallelism because metadata can now be moved into `PreloadedMetadata` before execution and Rust execution runs under `py.allow_threads`.

However, the current executor is still serial and mutable-state-centric.

The correct path is:

```text
Issue 1:
    Build safe architecture first.

Issue 2:
    Add controlled parallel ready-node execution.

Issues 3-6:
    Add independent performance optimizations after correctness foundation exists.
```

Do not start by wrapping the whole executor in:

```rust
Arc<Mutex<ExecutionContext>>
```

Do not parallelize sequential groups initially.

Do not allow worker execution to call Python.

Do not change calculation semantics or output format as part of these issues.



# Additional Omni-Calc Performance Issues: Data Structures, Clone Reduction, and Preloaded Data Usage

## Context

Branch:

```text
Blox-Dev / BLOX-2104-improve-preload-snapshot-py-rust-pass
```

GitHub:

```text
https://github.com/BloxSoftware/Blox-Dev/tree/BLOX-2104-improve-preload-snapshot-py-rust-pass
```

These issues are additional performance/refactor issues after the first 6 parallelization issues.

The first 6 issues mainly cover:

```text
Issue 1 - Kahn-style scheduler foundation
Issue 2 - Parallel ready-node execution with Rayon
Issue 3 - Parallel final RecordBatch materialization
Issue 4 - Parallel connected dimension preload
Issue 5 - Parallel join-path creation/alignment
Issue 6 - Parallel lookup-map aggregation
```

The following issues focus on improving performance by:

```text
better data structures
faster column lookup
less cloning
reusing preloaded data
avoiding Python callbacks in execution hot path
making snapshots/evaluators/resolvers more reference-based
```

Important note:

```text
PyO3 should remain only at the Python boundary / preload stage.

After metadata is preloaded into Rust, Omni-Calc execution should use Rust-owned preloaded data and should not call back into Python from the execution hot path.
```

---

# ISSUE 7

## Title

Optimize `CalcObjectState` Column Storage and Lookup With Ordered Indexed `ColumnStore`

## Context / Problem

The current executor stores block and dimension columns in `CalcObjectState` as ordered vectors:

```rust
Vec<(String, Vec<f64>)>
Vec<(String, Vec<String>)>
```

Current `CalcObjectState` fields are still:

```rust
pub dim_columns: Vec<(String, Vec<String>)>,
pub number_columns: Vec<(String, Vec<f64>)>,
pub string_columns: Vec<(String, Vec<String>)>,
pub connected_dim_columns: Vec<(String, Vec<String>)>,
```

This preserves deterministic output order, but lookup is linear. Current helper methods use patterns like:

```rust
self.number_columns
    .iter()
    .find(|(n, _)| n == name)
```

Similar linear checks exist in calculation-step setup, connected-dimension preload, sequential dependency collection, duplicate handling, and join path creation.

This becomes expensive for wide models with many indicators, properties, connected dimensions, cross-object columns, or repeated formula dependency lookups.

## Goal

Introduce a reusable ordered indexed column container that keeps the existing output order while making lookup, duplicate detection, insertion, and replacement faster.

This is the foundation for later shared `Arc` storage, clone reduction, FormulaEvaluator refactoring, interning, typed IDs, and future scheduler/merge work.

## Current Gap In Code

Current code still has:

```text
linear column lookup
linear duplicate checks before insertion
linear scans in connected dimension preload
linear scans when merging resolved columns
linear scans in FormulaEvaluator setup
linear scans in join path construction
```

There is no production `ColumnStore<T>` used by `CalcObjectState`.

There are early `calc_buffers` structures, but they are not integrated with the current `exec` hot path and should not be treated as the completed solution.

## Required Changes

### 1. Add `ColumnStore<T>`

Add an ordered indexed storage type, preferably under `src/engine/exec/column_store.rs` or another shared execution module.

Initial version should remain string-keyed for compatibility:

```rust
pub struct ColumnStore<T> {
    columns: Vec<(String, Vec<T>)>,
    index: HashMap<String, usize>,
}
```

Required methods:

```rust
new()
with_capacity(...)
insert(name, values)
replace(name, values)
insert_or_replace(name, values)
get(name)
get_mut(name)
contains(name)
remove(name)
iter()
iter_mut()
len()
is_empty()
column_names()
into_vec()
from_vec(...)
```

Behavior requirements:

```text
Preserve insertion order.
Prevent accidental duplicate insertion by default.
Allow explicit replacement where current behavior requires replacement.
Keep external string column names unchanged.
Support deterministic RecordBatch field order.
```

### 2. Migrate `CalcObjectState` Dynamic Columns

Migrate these first:

```text
number_columns
string_columns
connected_dim_columns
```

Keep `dim_columns` as `Vec<(String, Vec<String>)>` initially if changing it creates too much churn. Once dynamic columns are stable, evaluate migrating `dim_columns` too.

### 3. Preserve Compatibility Helpers

Keep methods such as:

```rust
get_number_column(...)
get_string_column(...)
get_connected_dim_column(...)
add_number_column(...)
add_string_column(...)
add_connected_dim_column(...)
```

but make them use `ColumnStore` internally.

### 4. Update RecordBatch Materialization

Update `build_record_batch` / `build_record_batch_with_timing` to iterate through `ColumnStore` in stable insertion order.

The final Arrow schema order must stay compatible with current output:

```text
dim columns
connected dimension columns
string columns
number columns
```

### 5. Update Hot-Path Insert/Contains Call Sites

Replace repeated patterns like:

```rust
state.number_columns.iter().any(|(n, _)| n == &col_name)
```

with:

```rust
state.number_columns.contains(&col_name)
```

Target call sites include:

```text
executor.rs calculation step column merge
executor.rs sequential step dependency merge
executor.rs connected dimension preload
context.rs RecordBatch duplicate checks
steps/calculation.rs existing column lookup
resolver/join preparation where state columns are searched
```

### 6. Add Benchmarks / Runtime Checks

Add or update focused tests/benchmarks for:

```text
wide column lookup
duplicate insert detection
stable iteration order
RecordBatch schema order
serial output parity
```

## Dependencies

```text
Must be implemented before Source Issue 11.
Source Issue 11 should build Arc-backed storage on top of this structure.
Source Issue 8 and Source Issue 19 depend on this for clone reduction.
```

This issue does not require Kahn scheduler or Rayon execution.

## Risks / Tradeoffs

```text
1. Many call sites currently expect Vec<(String, Vec<T>)>.
2. Borrowing may be trickier when state is mutated and read in the same function.
3. Output schema order must not change.
4. Some paths intentionally replace columns, especially sequential property handling.
5. A compatibility conversion layer may be needed during migration.
```

## Acceptance Criteria

```text
1. ColumnStore<T> exists with indexed lookup and stable iteration order.
2. CalcObjectState uses ColumnStore for number_columns.
3. CalcObjectState uses ColumnStore for string_columns.
4. CalcObjectState uses ColumnStore for connected_dim_columns.
5. Existing helper methods continue to work.
6. Number/string/connected column lookup no longer scans vectors.
7. Duplicate detection uses the index where ColumnStore is active.
8. RecordBatch output schema order remains deterministic and compatible.
9. Serial omni-calc output matches pre-refactor output.
10. Tests cover insert, replace, duplicate handling, get, contains, remove, and order.
11. Benchmarks or runtime metrics show lookup/duplicate-check cost is reduced or unchanged.
```

---

# ISSUE 8

## Title

Reduce Clone-Heavy Execution Paths by Reusing Shared Columns, Resolver Data, and Preloaded Metadata

## Context / Problem

After Source Issue 7 and Source Issue 11 introduce indexed and shared column storage, the executor should stop cloning large data structures unnecessarily.

Current clone-heavy areas include:

```text
ExecutionContext::new clones node_maps and variable_filters into CrossObjectResolver.
build_record_batch clones column vectors into Arrow arrays.
build_execution_result clones warnings into result.
FormulaEvaluator setup clones existing number columns.
FormulaEvaluator setup clones dimension/time string columns.
with_raw_properties clones EvalContext.
Resolver update rebuilds RecordBatch from state.
Resolver extracts RecordBatch columns into owned Vecs.
Connected dimension and sequential paths clone dimension/connected columns.
Property caches are mutable HashMaps on ExecutionContext.
```

The branch already has `PreloadedMetadata`, but it is not yet shared through `Arc`, not fully normalized for all worker execution needs, and not used as a complete immutable execution snapshot.

## Goal

Reduce CPU and memory overhead from unnecessary cloning while preserving current calculation behavior and output format.

Use shared immutable data wherever possible, and clone only when ownership is required for newly calculated outputs or final API materialization.

## Current Gap In Code

The current code partially moved property loading toward preloaded metadata, but several legacy clone-heavy patterns remain.

Current `Engine` stores:

```rust
preloaded_metadata: Option<PreloadedMetadata>
```

and exposes:

```rust
Option<&PreloadedMetadata>
```

This avoids some copying, but it is not yet the `Arc<PreloadedMetadata>` / immutable snapshot shape needed for cheap worker snapshots.

Legacy Python metadata loaders still exist and must remain isolated from optimized execution paths.

## Required Changes

### 1. Use Shared Column Storage From Issues 7 and 11

Replace clone-heavy `Vec<T>` movement with shared columns where possible:

```text
formula input columns
string dimension columns
time values
connected dimension columns
resolver source columns
snapshot columns
```

### 2. Store PreloadedMetadata as Shared Read-Only Data

Move toward:

```rust
Arc<PreloadedMetadata>
```

or an equivalent immutable shared snapshot.

Avoid copying these maps:

```rust
HashMap<i64, Vec<DimensionItem>>
HashMap<(i64, i64, i64), HashMap<i64, String>>
```

### 3. Build Immutable Property Cache Snapshots

Current execution has mutable caches:

```rust
string_property_map_cache
numeric_property_map_cache
```

Introduce a read-only structure:

```rust
struct PropertyCacheSnapshot {
    string_maps: HashMap<(i64, i64, i64), Arc<StringPropertyMap>>,
    numeric_maps: HashMap<(i64, i64, i64), Arc<PropertyMap>>,
}
```

Build from `PreloadedMetadata`.

Do not call Python/PyO3 in optimized execution paths.

### 4. Reduce Resolver Materialization Clones

Current resolver stores `RecordBatch` data and extracts columns back into owned vectors.

Move toward storing a lighter shared block snapshot:

```rust
struct BlockSnapshot {
    dim_columns: Arc<ColumnStore<String>>,
    connected_dim_columns: Arc<ColumnStore<String>>,
    string_columns: Arc<ColumnStore<String>>,
    number_columns: Arc<ColumnStore<f64>>,
}
```

Intermediate step is acceptable: reduce how often full `RecordBatch` rebuild happens and track remaining materialization boundaries.

### 5. Avoid Re-Cloning Plan Maps Where Possible

Current resolver construction clones:

```rust
plan.request.node_maps.clone()
plan.request.variable_filters.clone()
```

Use references or `Arc` if lifetime complexity is acceptable:

```rust
Arc<[PlannedNodeMap]>
Arc<HashMap<String, VariableFilter>>
```

or a borrowed resolver shape tied to the plan lifetime.

### 6. Track Remaining Clone Boundaries

Keep or add performance counters for:

```text
number column clone count
string column clone count
RecordBatch materialization count
formula context clone count
warning clone count
estimated clone bytes
```

## Dependencies

```text
Depends on Source Issue 7.
Depends on Source Issue 11.
Strongly related to complete preload normalization in Source Issue 10 / scheduler foundation.
Feeds Source Issue 19 FormulaEvaluator refactor.
```

## Risks / Tradeoffs

```text
1. Some clones are still required for newly calculated outputs.
2. Arrow materialization may still copy; full Arrow zero-copy is out of scope.
3. Arc-based ownership can make replacement semantics more explicit.
4. Resolver behavior is correctness-sensitive.
5. Moving caches to immutable snapshots may expose missing preload gaps.
```

## Acceptance Criteria

```text
1. Major clone-heavy paths are documented and measured.
2. PreloadedMetadata is shared by reference/Arc and not repeatedly cloned.
3. Property maps are built from PreloadedMetadata, not Python callbacks.
4. Formula input setup uses shared columns where possible.
5. Resolver update/materialization avoids unnecessary full data cloning where possible.
6. RecordBatch materialization remains correct and isolated to needed boundaries.
7. Existing output schema and ordering remain unchanged.
8. Serial executor output matches current behavior.
9. No PyO3/Python callbacks are introduced in execution hot paths.
10. Clone counters or benchmarks show reduced cloning, or remaining clone boundaries are explicitly documented.
```

---

# Coverage Check: Do Issues 1-9 Cover Preloaded Data and Python Callback Removal?

## Short Answer

Yes, the full issue set from Issue 1 to Issue 9 covers this.

The intended rule is:

```text
Use PyO3 only at the Python boundary and preload stage.
Do not call Python/PyO3 from the Rust execution hot path after preload.
Use PreloadedMetadata and immutable Rust-owned snapshots during execution.
```

---

## Where It Is Covered

### Issue 1

Covers:

```text
Remove/isolate Python callback paths from Rust hot path
Preloaded-only worker-ready execution
No Python callbacks during node execution
Fail early if required preloaded metadata is missing
```

### Issue 2

Covers:

```text
Parallel workers must not call Python
Parallel workers must use immutable snapshots
Parallel property/input/calculation execution must use preloaded Rust data
```

### Issue 7

Covers:

```text
Better state structure for faster access to already-loaded data
Faster merge and lookup without repeated scans
Improves use of Rust-owned execution state
```

### Issue 8

Covers:

```text
Reuse preloaded data by reference or Arc
Avoid cloning preloaded maps and large column vectors
Use shared column references in snapshots/evaluators/resolver
Avoid Python fallback in optimized execution paths
```

### Issue 9

Covers separately:

```text
Avoid repeated String allocation/cloning for join keys
Improve cross-object join key representation
Make better use of IDs / encoded keys instead of repeated string paths
```

---

## Final Recommended Rule To Add Across Issues

Add this sentence to all implementation tickets touching execution:

```text
All execution-hot-path logic must use Rust-owned preloaded data and must not perform PyO3/Python metadata callbacks. PyO3 is allowed only at the external Python binding boundary and during the explicit preload phase before Rust execution starts.
```

---

# Final Notes

The first 6 issues cover the parallelization strategy.

Issue 7 and Issue 8 add the missing performance foundation around:

```text
better data structures
faster column lookup
less cloning
better reuse of preloaded data
no Python callbacks in execution
lower memory pressure
```

Together, Issues 1-9 give a stronger optimization plan:

```text
Issues 1-6:
    safe parallelism

Issue 7:
    faster state and column lookup

Issue 8:
    less cloning and better shared data reuse

Issue 9:
    faster join keys and less string allocation
```


# Epic: Performance, Preloading, Data Structure, and Parallel Execution Improvements for Rust `omni-calc`

## Summary

This epic covers additional performance and architecture improvements for the Rust `omni-calc` executor after the initial executor refactor work.

The goal is to improve:

- Better use of `PreloadedMetadata`
- Safer future Rayon-based parallel execution
- Reduced Python callback dependency
- Faster snapshot creation
- Lower memory cloning overhead
- Faster resolver refresh behavior
- Better data structures for hot execution paths
- Improved dependency validation before parallel execution
- Benchmarking and profiling coverage

These issues should be treated as follow-up tickets after the initial Issues 1–9 executor refactor work.

The main direction is:

```text
Python = plan creation and metadata preparation
Rust = execution, dependency tracking, snapshots, scheduling, and merge
Workers = read immutable snapshots only
Merge phase = only place where shared state is mutated
```

Python callbacks should not be introduced inside worker-style execution paths.

---

# Issue 10: Expand and Normalize `PreloadedMetadata` for Worker-Safe Execution

## Summary

Improve `PreloadedMetadata` so Rust execution workers can read all required metadata from Rust-owned immutable structures without needing Python callbacks, lazy loading, or shared mutable metadata caches during calculation.

## Problem

For future Rayon-based parallel execution, worker threads should not depend on:

- Python callbacks
- GIL-bound metadata access
- Shared mutable metadata caches
- Runtime lazy metadata resolution
- Blocking metadata fetches during node execution

If metadata is incomplete during execution, parallel execution can become blocked, non-deterministic, or slower than serial execution.

## Proposed Direction

Refactor and expand `PreloadedMetadata` to include all metadata required by:

- Input step execution
- Calculation step execution
- Property step execution
- Cross-object resolution
- Variable filters
- Dimension lookups
- Property map lookups
- Formula dependency handling
- Source block / connected block resolution

The metadata should be loaded before execution starts and shared through immutable structures.

Example direction:

```rust
struct PreloadedMetadata {
    blocks: Arc<HashMap<BlockId, BlockMetadata>>,
    indicators: Arc<HashMap<IndicatorId, IndicatorMetadata>>,
    dimensions: Arc<HashMap<DimensionId, DimensionMetadata>>,
    properties: Arc<HashMap<PropertyId, PropertyMetadata>>,
    variable_filters: Arc<HashMap<FilterId, VariableFilterMetadata>>,
    property_maps: Arc<PropertyMapStore>,
}
```

Execution workers should receive metadata through `ExecutionSnapshot`.

```rust
struct ExecutionSnapshot {
    block_id: BlockId,
    preloaded_metadata: Arc<PreloadedMetadata>,
    resolver_snapshot: Arc<CrossObjectResolverSnapshot>,
}
```

## Acceptance Criteria

- Required execution metadata is available before node execution starts.
- Worker-style execution paths do not call back into Python.
- Metadata is stored in immutable, shareable Rust structures such as `Arc`.
- Existing metadata caches are removed from hot paths where possible.
- Missing metadata produces a clear validation error before execution starts.
- Tests verify that execution works using only preloaded metadata.
- No new Python callback is introduced inside the Rust node execution path.

---

# ISSUE 11

## Title

Introduce Snapshot-Friendly Shared Column Storage Using `Arc` on Top of `ColumnStore`

## Context / Problem

Future Kahn-style scheduling and Rayon execution require read-only execution snapshots. If snapshots deep-clone every column, then snapshot creation can become slower and more memory-heavy than current serial execution.

Current column data is still owned directly as:

```rust
Vec<f64>
Vec<String>
```

and columns are copied in several places:

```text
RecordBatch materialization clones vectors into Arrow arrays.
FormulaEvaluator setup clones existing columns.
Resolver update rebuilds RecordBatch snapshots.
Sequential and calculation steps clone dimension/connected columns.
```

Source Issue 11 should be solved after Source Issue 7 because shared storage needs a stable indexed container to live in.

## Goal

Make column storage snapshot-friendly by allowing existing columns to be shared cheaply through immutable `Arc` data.

Snapshots should be cheap to clone, and only newly calculated outputs should be owned until they are merged into shared state.

## Current Gap In Code

The current code does not have production shared column storage in `CalcObjectState`.

Some Arrow `ArrayRef` and `calc_buffers` structures exist, but the main executor still uses `Vec<(String, Vec<T>)>` and clones data in hot paths.

## Required Changes

### 1. Add Shared Column Aliases

Preferred long-term aliases:

```rust
pub type SharedNumberColumn = Arc<[f64]>;
pub type SharedStringColumn = Arc<[String]>;
```

Acceptable first step if conversion churn is too high:

```rust
pub type SharedNumberColumn = Arc<Vec<f64>>;
pub type SharedStringColumn = Arc<Vec<String>>;
```

Prefer `Arc<[T]>` because it clearly represents immutable shared slice data.

### 2. Upgrade ColumnStore Value Storage

Add a shared variant or evolve `ColumnStore<T>`:

```rust
pub struct SharedColumnStore<T> {
    columns: Vec<(String, Arc<[T]>)>,
    index: HashMap<String, usize>,
}
```

or:

```rust
pub struct ColumnStore<T> {
    columns: Vec<(String, SharedColumn<T>)>,
    index: HashMap<String, usize>,
}
```

The design must support:

```text
stable order
fast lookup
cheap clone of column references
explicit replacement
owned output conversion after node calculation
```

### 3. Define Snapshot Shapes

Introduce or prepare a snapshot-friendly shape:

```rust
struct ExecutionSnapshot {
    block_key: String,
    dim_columns: Arc<ColumnStore<String>>,
    number_columns: Arc<ColumnStore<f64>>,
    string_columns: Arc<ColumnStore<String>>,
    connected_dim_columns: Arc<ColumnStore<String>>,
    preloaded_metadata: Arc<PreloadedMetadata>,
}
```

This issue does not need to implement the full scheduler, but it must make column state compatible with that model.

### 4. Keep Newly Calculated Outputs Owned Until Merge

New node outputs should still be owned while being produced:

```rust
struct NodeOutput {
    block_key: String,
    number_columns: Vec<(String, Vec<f64>)>,
    string_columns: Vec<(String, Vec<String>)>,
    connected_dim_columns: Vec<(String, Vec<String>)>,
}
```

After merge, convert owned vectors into shared columns.

### 5. Prepare RecordBatch Materialization

`build_record_batch` may still need to create Arrow arrays, but it should read from shared slices instead of forcing earlier clones.

If Arrow conversion still copies, keep that cost isolated to final materialization and resolver boundaries until the resolver is refactored.

## Dependencies

```text
Depends on Source Issue 7 ColumnStore.
Enables Source Issue 8 clone reduction.
Enables Source Issue 19 FormulaEvaluator shared context.
Supports later scheduler/snapshot work.
```

## Risks / Tradeoffs

```text
1. Arc-backed data is immutable, so mutation must become append/replace, not in-place editing.
2. Some existing code expects owned Vec<T>.
3. Temporary compatibility conversions may hide clone costs if not tracked.
4. Arc<Vec<T>> is easier but less semantically clean than Arc<[T]>.
5. Full Arrow zero-copy is out of scope.
```

## Acceptance Criteria

```text
1. Shared column aliases exist.
2. ColumnStore can store or expose shared column data.
3. Existing columns can be shared without deep cloning.
4. Snapshot-style structures can clone references cheaply.
5. Newly calculated outputs remain owned until merge.
6. RecordBatch schema/order remains unchanged.
7. Serial executor output remains identical.
8. Tests prove shared columns preserve lookup and ordering behavior.
9. Tests prove snapshot-style clone does not deep-clone column vectors.
10. Benchmarks or counters show reduced snapshot/column clone overhead or document remaining clone boundaries.
```

---

# Issue 12: Add Rayon-Based Ready-Layer Parallel Execution Behind a Feature Flag

## Summary

Add an experimental Rayon-based execution path that parallelizes only safe ready nodes after the single-threaded Kahn-style scheduler has been introduced and validated.

## Problem

The executor currently runs serially.

Even after graph-driven scheduling is introduced, execution will remain single-threaded unless ready nodes are executed concurrently.

However, parallelization should not be added blindly to all nodes because some nodes may depend on:

- Sequential execution semantics
- Resolver update timing
- Cross-object references
- Python-backed metadata
- Shared mutable state
- Overlapping output columns

## Proposed Direction

Add a feature-gated or config-gated parallel execution mode.

Example:

```rust
let outputs: Vec<NodeOutput> = ready_nodes
    .into_par_iter()
    .map(|node| {
        let snapshot = build_snapshot_readonly(&ctx, &node);
        execute_node(snapshot, node)
    })
    .collect();

merge_node_outputs(&mut ctx, outputs);
```

Only nodes marked as parallel-safe should be included.

```rust
struct ExecNode {
    id: NodeId,
    block_id: BlockId,
    deps: Vec<NodeId>,
    outputs: Vec<ColumnId>,
    parallel_safe: bool,
}
```

## Safe Initial Candidates

Parallelization can initially be allowed for:

- Independent input indicator nodes
- Calculation nodes whose dependencies are complete
- Property nodes that only read preloaded metadata
- Nodes that produce distinct output columns
- Nodes that do not require immediate resolver mutation

## Do Not Parallelize Initially

Do not parallelize these node types initially:

- Sequential groups
- Nodes requiring unresolved cross-object references
- Nodes requiring Python callbacks
- Nodes depending on immediate mutable resolver updates
- Nodes writing to overlapping output columns
- Nodes with unclear dependency boundaries

## Acceptance Criteria

- Parallel execution is behind a feature flag or config option.
- Default execution remains serial unless parallel mode is explicitly enabled.
- Rayon is used only for ready nodes marked as parallel-safe.
- Sequential groups continue to execute atomically.
- Output ordering remains deterministic where required.
- Serial and parallel execution outputs are tested for equality.
- Tests verify unsafe nodes are excluded from Rayon execution.
- Benchmarks show speedup on eligible workloads.

---

# ISSUE 13

## Title

Replace Selected Hot-Path String Lookups With Interned IDs

## Context / Problem

Current execution uses many repeated string identifiers:

```text
b123
ind456
_789
prop123
block123___ind456
dim123___prop456
join path strings
variable names
node map keys
```

These strings are cloned, hashed, compared, formatted, split, and parsed repeatedly during:

```text
calculation-step processing
cross-object resolver lookup
node map lookup
formula dependency handling
column lookup
join path creation
lookup map creation
warning/debug generation
future graph construction
```

Source Issue 13 is not about benchmarking. It is specifically about reducing hot-path string overhead using interned IDs.

## Goal

Introduce a lightweight string interning layer for selected high-volume keys while preserving external string names for API output, RecordBatch schema, warnings, errors, and debug logs.

This is an incremental optimization before full typed ID migration.

## Current Gap In Code

There is no production string interner.

Current code still uses:

```rust
HashMap<String, ...>
Vec<(String, ...)>
String join paths
String NodeMapKey fields
String cross-object references
```

Some typed wrappers exist, but they still wrap `String` and do not eliminate hot-path string parsing/hash cost.

## Required Changes

### 1. Add String Interner Types

Add types such as:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct InternedId(u32);

pub struct StringInterner {
    strings: Vec<String>,
    index: HashMap<String, InternedId>,
}
```

Required methods:

```rust
intern(&mut self, value: &str) -> InternedId
resolve(&self, id: InternedId) -> Option<&str>
contains(value)
len()
```

### 2. Intern Selected Keys During Plan Preparation

Intern keys once before hot execution:

```text
block keys
node IDs
indicator column names
property column names
dimension column names
cross-object references
variable names
node map key components
```

Do not intern every string in the system in the first PR.

### 3. Use Interned IDs In Narrow Hot Paths First

Good initial candidates:

```text
ColumnStore index keys
NodeMapKey lookup
resolver calculated block lookup
formula existing-column lookup
step node ID classification cache
```

Keep a string compatibility layer at boundaries.

### 4. Preserve External Names

Original names must remain available for:

```text
RecordBatch field names
Python/API output
warnings
errors
debug logs
formula strings
```

### 5. Add Instrumentation / Tests

Track whether interning reduces:

```text
string allocation
string clone count
HashMap<String> lookup cost
resolver lookup time
column lookup time
```

## Dependencies

```text
Should be implemented after Source Issue 7.
Works better after Source Issue 11 and Source Issue 8.
Can be implemented before Source Issue 21 as a bridge toward typed IDs.
Does not require full scheduler work.
```

## Risks / Tradeoffs

```text
1. Over-interning can add complexity without measurable gain.
2. Interned IDs must not leak into external output.
3. Debug/error messages must remain readable.
4. Interner lifetime must cover the execution run.
5. Multiple interner instances can cause invalid ID comparisons if not scoped carefully.
6. This is not a replacement for semantic typed IDs.
```

## Acceptance Criteria

```text
1. StringInterner and InternedId exist.
2. Interning is applied to at least one measured hot path.
3. Original string names are preserved for output/debug/error boundaries.
4. Tests cover intern, resolve, duplicate intern, missing resolve, and stable IDs.
5. Tests confirm output column names remain unchanged.
6. Benchmarks or runtime metrics show improvement or document neutral result.
7. Migration plan identifies remaining string-heavy paths for Source Issue 21.
```

---

# Issue 14: Add Dependency-Aware Resolver Snapshot Refresh

## Summary

Optimize `CrossObjectResolver` updates by refreshing resolver snapshots only when downstream dependencies require updated source block data.

## Problem

Currently, resolver updates may rebuild a full `RecordBatch` after individual node execution.

This can cause repeated expensive work.

Current behavior can look like:

```text
node 1 -> rebuild source block RecordBatch
node 2 -> rebuild source block RecordBatch
node 3 -> rebuild source block RecordBatch
```

For large blocks, repeated Arrow array construction and cloning can be costly.

## Proposed Direction

Track which downstream nodes require resolver-visible data from each source block.

Only refresh resolver state when:

- A source block has completed the required columns for downstream references
- A dependency boundary is reached
- A ready batch/layer has finished
- A downstream node is about to read cross-object data
- Resolver-visible data has actually changed

Example concept:

```rust
struct ResolverRefreshPlan {
    required_by_block: HashMap<BlockId, Vec<CrossObjectDependency>>,
    dirty_blocks: HashSet<BlockId>,
}
```

Resolver updates should happen in a controlled merge phase, not inside each node execution.

## Acceptance Criteria

- Resolver refresh is not blindly triggered after every node output.
- Resolver update decisions are dependency-aware.
- Cross-object references still resolve correctly.
- Serial behavior remains unchanged.
- Resolver is updated before downstream nodes that need source data are released.
- Tests cover same-block dependencies.
- Tests cover cross-block dependencies.
- Tests cover resolver refresh timing.
- Tests cover missing source data errors.
- Performance improves for plans with multiple outputs from the same source block.

---

# Issue 15: Batch `NodeOutput` Merge Operations

## Summary

Improve merge performance by batching multiple `NodeOutput` values before mutating `ExecutionContext`.

## Problem

In a future parallel ready-layer execution model, multiple nodes may finish together and produce multiple `NodeOutput` values.

Merging each output one by one can cause repeated:

- Block state lookups
- Column existence checks
- Resolver refresh checks
- Warning extensions
- Stats updates
- Mutable state operations

## Proposed Direction

Introduce a batched merge function.

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

- `block_id`
- affected output columns
- resolver update requirement
- warning collection
- calculated node count

Example internal grouping:

```rust
struct BlockOutputBatch {
    block_id: BlockId,
    number_columns: Vec<(ColumnId, Vec<f64>)>,
    string_columns: Vec<(ColumnId, Vec<String>)>,
    connected_dim_columns: Vec<(ColumnId, Vec<String>)>,
    should_update_resolver: bool,
}
```

## Acceptance Criteria

- Multiple `NodeOutput` values can be merged in one controlled mutation phase.
- Outputs are grouped by block before merge.
- Warnings and stats are accumulated once per batch.
- Resolver refresh happens at most once per affected block per merge batch.
- Existing serial behavior remains unchanged.
- Tests cover multiple outputs for the same block.
- Tests cover multiple outputs for different blocks.
- Tests cover duplicate column prevention.
- Tests cover resolver update behavior during batch merge.

---

# Issue 16: Add Execution Profiling and Benchmarks for Scheduler, Snapshot, Merge, and Resolver

## Summary

Add benchmark and profiling coverage for the new execution architecture to measure real performance gains and identify bottlenecks.

## Problem

Executor refactors around snapshots, preloading, graph scheduling, resolver updates, Arrow `RecordBatch` construction, and Rayon parallelism need measurable validation.

Without benchmarks, it is difficult to know whether changes actually improve performance or only improve architecture.

## Proposed Direction

Add benchmarks for:

- Graph construction
- Formula dependency extraction
- Metadata preload
- Snapshot creation
- Node execution
- `NodeOutput` merge
- Batched merge
- Resolver update
- Arrow `RecordBatch` construction
- Serial Kahn scheduler
- Parallel ready-layer execution
- Large block execution
- Cross-object dependency-heavy execution
- Property-heavy execution
- Sequential group execution

Suggested benchmark categories:

```text
Small plan:
  low number of blocks and indicators

Medium plan:
  multiple blocks, formulas, properties, and filters

Large plan:
  large column vectors and many calculation nodes

Cross-object plan:
  many source block references

Property-heavy plan:
  many metadata/property lookups
```

## Acceptance Criteria

- Benchmark cases exist for small, medium, and large plans.
- Resolver rebuild cost is measured separately.
- Snapshot creation cost is measured separately.
- Serial vs parallel execution can be compared.
- Metadata preload cost is measured separately.
- Arrow `RecordBatch` construction cost is measured separately.
- Benchmarks can be run locally by developers.
- Results help identify remaining hot paths.
- Documentation explains how to run the benchmarks.

---

# Issue 17: Add Pre-Execution Validation for Parallel-Safety

## Summary

Add a validation pass that checks whether each execution node is safe for future parallel execution.

## Problem

Not every node should be parallelized.

Some nodes may depend on:

- Sequential semantics
- Mutable resolver state
- Cross-object data availability
- Python-backed metadata access
- Overlapping output columns
- Missing dependency information
- Shared mutable caches

Without validation, parallel execution may produce incorrect or non-deterministic results.

## Proposed Direction

Add a pre-execution validation phase that marks each node as either parallel-safe or not parallel-safe.

Example:

```rust
struct ParallelSafety {
    parallel_safe: bool,
    reasons: Vec<ParallelSafetyReason>,
}
```

Example reasons:

```rust
enum ParallelSafetyReason {
    SequentialGroup,
    RequiresPythonCallback,
    RequiresMutableResolverDuringExecution,
    HasUnresolvedCrossObjectDependency,
    WritesOverlappingOutputColumn,
    MissingPreloadedMetadata,
    UnknownDependencyBoundary,
}
```

Each `ExecNode` should include the validation result:

```rust
struct ExecNode {
    id: NodeId,
    block_id: BlockId,
    deps: Vec<NodeId>,
    outputs: Vec<ColumnId>,
    parallel_safety: ParallelSafety,
}
```

## Validation Should Check

- Whether all metadata is preloaded
- Whether dependencies are explicit
- Whether outputs are distinct
- Whether the node belongs to a sequential group
- Whether it requires resolver updates before downstream execution
- Whether it depends on Python callbacks
- Whether it mutates shared state
- Whether output columns overlap with another ready node

## Acceptance Criteria

- Every node has a clear parallel-safety classification.
- Unsafe nodes are excluded from Rayon execution.
- Validation errors are clear and actionable.
- Sequential nodes are always marked unsafe initially.
- Missing metadata marks the node as unsafe or fails before execution.
- Tests cover safe node classification.
- Tests cover unsafe node classification.
- Tests cover overlapping output columns.
- Tests cover Python callback dependency detection.
- Tests cover sequential group safety handling.

---

# Issue 18: Optimize Formula Dependency Parsing and Graph Build

## Summary

Improve formula dependency extraction so the Rust execution graph can be built accurately and efficiently without repeatedly parsing formulas during scheduling or execution.

## Problem

A future Rust-side Kahn scheduler requires accurate dependency tracking between nodes.

If formula references are parsed repeatedly during execution, graph construction and scheduling can become slower and more error-prone.

Formula dependency parsing is especially important for references like:

```text
ind259068
block39951___ind259068
property references
variable filter references
connected block references
```

## Proposed Direction

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

## Acceptance Criteria

- Formula dependencies are parsed once during graph construction.
- Same-block dependencies are represented explicitly.
- Cross-object dependencies are represented explicitly.
- Property dependencies are represented explicitly.
- Scheduler does not parse formula strings repeatedly.
- Dependency extraction handles same-block indicator references.
- Dependency extraction handles cross-block indicator references.
- Dependency extraction handles property references.
- Dependency extraction handles sequential group dependencies.
- Tests cover dependency parsing.
- Tests cover graph ordering.
- Tests cover missing dependency detection.
- Tests cover invalid formula reference errors.

---

# Recommended Priority for Issues 10–18

Suggested implementation order:

1. Issue 10: Expand and Normalize `PreloadedMetadata` for Worker-Safe Execution
2. Issue 11: Introduce Snapshot-Friendly Column Storage Using Shared `Arc` Data
3. Issue 14: Add Dependency-Aware Resolver Snapshot Refresh
4. Issue 15: Batch `NodeOutput` Merge Operations
5. Issue 17: Add Pre-Execution Validation for Parallel-Safety
6. Issue 18: Optimize Formula Dependency Parsing and Graph Build
7. Issue 12: Add Rayon-Based Ready-Layer Parallel Execution Behind a Feature Flag
8. Issue 13: Replace Hot-Path String Lookups with Interned IDs
9. Issue 16: Add Execution Profiling and Benchmarks for Scheduler, Snapshot, Merge, and Resolver

---

# Overall Notes

These issues should not introduce Python callbacks inside Rust worker execution.

The preferred architecture is:

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

Avoid this pattern:

```rust
Arc<Mutex<ExecutionContext>>
```

for the full node execution lifecycle.

Prefer this pattern:

```text
Build immutable snapshot
        ↓
Execute node without mutating global state
        ↓
Return NodeOutput
        ↓
Batch merge outputs in controlled mutation phase
        ↓
Refresh resolver only when needed
        ↓
Release dependent nodes
```

This gives the executor a safe path toward real Rayon-based parallel execution without breaking current calculation semantics.



# Additional Omni-Calc Performance Issues

Branch:

```text
Blox-Dev / BLOX-2104-improve-preload-snapshot-py-rust-pass
```

GitHub:

```text
https://github.com/BloxSoftware/Blox-Dev/tree/BLOX-2104-improve-preload-snapshot-py-rust-pass
```

These issues are additional follow-up issues after the existing 18 Omni-Calc performance / parallelization issues.

They focus on deeper Rust performance improvements around:

```text
formula evaluator cloning
shared evaluator context
actuals / forecast handling
row key reuse
structured IDs instead of repeated string parsing
better internal data structures
less cloning
better reuse of preloaded data
faster Omni-Calc execution
```

Important global rule for all issues:

```text
All Omni-Calc execution-hot-path logic should use Rust-owned preloaded data and should not perform PyO3/Python metadata callbacks after preload.

PyO3 should remain only at:
1. Python binding boundary
2. CalcPlan extraction
3. explicit metadata preload before Rust execution starts
4. returning results back to Python
```

---

# ISSUE 19

## Title

Optimize `FormulaEvaluator` Context Cloning and Shared Column Access

## Context / Problem

`FormulaEvaluator` is a core execution hot path. Current evaluator state still owns cloned column data:

```rust
HashMap<String, Vec<f64>>
HashMap<String, Vec<String>>
Vec<String>
```

Current `EvalContext` contains:

```rust
columns: HashMap<String, Vec<f64>>,
dim_string_columns: HashMap<String, Vec<String>>,
time_values: Vec<String>,
item_date_ranges: HashMap<(i64, String), ItemDateRange>,
```

Current `with_raw_properties()` creates a child evaluator by cloning the full `EvalContext`:

```rust
ctx: self.ctx.clone()
```

This happens for functions such as:

```text
max
min
mod
if / ifelse
```

Calculation setup also clones existing columns into the evaluator:

```rust
evaluator.add_column(name.clone(), values.clone())
```

Time and dimension string columns are also cloned during evaluator initialization.

## Goal

Split immutable formula input data from mutable evaluator state so child evaluators can share context cheaply.

Avoid cloning full numeric/string/time column maps during formula setup and raw-property child evaluation.

## Current Gap In Code

The current code added timing counters around formula parse/eval and raw-property clone count, but the underlying context still clones data.

Formula evaluation still returns owned `Vec<f64>`, which is acceptable for newly calculated outputs. The problem is cloning input context and source columns.

## Required Changes

### 1. Introduce Shared EvalContext Data

After Source Issues 7, 11, and 8, move evaluator input storage toward:

```rust
pub struct EvalContext {
    columns: HashMap<String, Arc<[f64]>>,
    dim_string_columns: HashMap<String, Arc<[String]>>,
    item_date_ranges: Arc<HashMap<(i64, String), ItemDateRange>>,
    time_values: Arc<[String]>,
    time_bounds: (String, String),
    integer_columns: HashSet<String>,
    row_count: usize,
}
```

If typed IDs or interned IDs are available later, the key can move from `String` to an internal ID. Do not block this issue on full typed ID migration.

### 2. Split Immutable Context From Mutable Evaluator State

Target shape:

```rust
pub struct FormulaEvaluator {
    ctx: Arc<EvalContext>,
    warnings: Vec<EvalWarning>,
    actuals_context: Option<Arc<ActualsContext>>,
    property_filter_context: PropertyFilterContext,
    prior_called: bool,
    last_result_is_integer: bool,
}
```

`warnings`, `prior_called`, `last_result_is_integer`, and filter mode remain evaluator-local mutable state.

### 3. Fix `with_raw_properties()` Clone Behavior

Replace full context clone with:

```rust
ctx: Arc::clone(&self.ctx)
```

The child evaluator should have:

```text
shared immutable input context
fresh warnings
same/Arc actuals context
property filtering disabled
reset prior_called
reset integer-result flag
```

### 4. Avoid Cloning Existing Columns Into Evaluator

Replace:

```rust
evaluator.add_column(name.clone(), values.clone())
```

with shared insertion:

```rust
evaluator.add_column_shared(name.clone(), Arc::clone(values))
```

or with a builder that receives shared `ColumnStore` references.

### 5. Keep Outputs Owned

Formula outputs are newly calculated and can remain:

```rust
Vec<f64>
```

After calculation, they may be added to shared column storage as `Arc<[f64]>` during merge.

### 6. Update Formula Functions To Prefer Slices

Where functions currently require `&Vec<T>`, update them to accept:

```rust
&[f64]
&[String]
```

This reduces pressure to clone just to satisfy function signatures.

### 7. Preserve Formula Semantics

Must preserve:

```text
property filtering behavior
raw property behavior inside IF/MAX/MIN/etc.
time functions
actuals context behavior
sequential function warnings
integer column tracking for days()/weekdays()
prior() zero-first-period behavior
```

## Dependencies

```text
Depends on Source Issue 7 for indexed/shared-compatible column access.
Depends on Source Issue 11 for Arc-backed columns.
Depends on Source Issue 8 for shared clone-reduction patterns.
Does not require Source Issue 13 or Source Issue 21.
```

## Risks / Tradeoffs

```text
1. Borrowed lifetime-based context is risky; use Arc first.
2. Some formula functions may need signature updates.
3. Mutable evaluator state must not be placed inside shared context.
4. Property filtering depends on evaluator-local mode and must remain correct.
5. ActualsContext may need Arc wrapping but must preserve behavior.
6. Existing tests may rely on exact warnings and integer handling.
```

## Acceptance Criteria

```text
1. FormulaEvaluator input context can be shared by Arc.
2. with_raw_properties() does not clone the full EvalContext.
3. Existing numeric input columns are not cloned into evaluator setup where shared storage is available.
4. Dimension/time string columns are not cloned into evaluator setup where shared storage is available.
5. Formula outputs remain correct and owned where necessary.
6. Property filtering behavior remains unchanged.
7. Raw property behavior inside IF/MAX/MIN/MOD remains unchanged.
8. Time functions and sequential-related formula behavior remain unchanged.
9. Existing formula, calculation, and sequential tests pass.
10. Runtime counters or benchmarks show reduced formula context clone cost.
```

---

# ISSUE 20

## Issue Type

Performance / Algorithm Optimization / Refactor

---

## Title

Optimize Actuals Handling, Forecast Indices, and Entity Key Reuse

---

## Summary

Optimize actuals and forecast-period handling in Omni-Calc by reducing repeated row-key construction, avoiding repeated string allocation, improving forecast membership checks, and reusing precomputed entity/dimension keys.

The current actuals path builds actual maps, forecast indices, dimension keys, and entity keys during calculation execution. This is correct but can be expensive for large blocks, many actuals-enabled indicators, or sequential formulas that need last-actual-by-entity logic.

This issue should introduce reusable per-block actuals/key metadata so repeated indicators do not rebuild the same row keys and entity keys again and again.

---

## Relevant Code Areas

```text
modelAPI/omni-calc/src/engine/exec/steps/calculation.rs
modelAPI/omni-calc/src/engine/exec/formula_eval.rs
modelAPI/omni-calc/src/engine/exec/steps/sequential.rs
modelAPI/omni-calc/src/engine/exec/state.rs
modelAPI/omni-calc/src/engine/integration/calc_plan.rs
```

Important current functions:

```rust
CalculationStepHandler::load_actuals_for_indicator(...)
CalculationStepHandler::build_dimension_key(...)
CalculationStepHandler::build_entity_key(...)
CalculationStepHandler::extract_last_actual_by_entity(...)
ActualsContext::is_forecast(...)
ActualsContext::get_last_actual(...)
```

Current actuals context:

```rust
pub struct ActualsContext {
    pub actual_values: Vec<f64>,
    pub forecast_indices: Vec<usize>,
    pub last_actual_by_entity: HashMap<String, f64>,
    pub has_actuals: bool,
}
```

Current forecast check:

```rust
pub fn is_forecast(&self, row_idx: usize) -> bool {
    self.forecast_indices.contains(&row_idx)
}
```

This is linear in `forecast_indices.len()`.

---

## Current Behavior

For each actuals-enabled indicator, the handler:

```text
1. builds an actual data map
2. computes forecast row indices
3. builds dimension keys for actual matching
4. builds entity keys for sequential/last-actual handling
5. extracts last actual value per entity
6. clones actual_values / forecast_indices into ActualsContext
```

Dimension row key creation:

```rust
fn build_dimension_key(
    &self,
    dim_columns: &HashMap<String, Vec<String>>,
    row_idx: usize,
) -> String {
    let mut key_parts: Vec<String> = Vec::new();

    for (col_name, values) in dim_columns {
        if let Some(dim_id) = col_name.strip_prefix('_') {
            if row_idx < values.len() {
                key_parts.push(format!("{}={}", dim_id, values[row_idx]));
            }
        }
    }

    key_parts.sort();
    key_parts.join("|")
}
```

Entity key creation:

```rust
fn build_entity_key(
    &self,
    dim_columns: &HashMap<String, Vec<String>>,
    time_col: Option<&str>,
    row_idx: usize,
) -> String {
    let mut key_parts: Vec<String> = Vec::new();

    for (col_name, values) in dim_columns {
        if Some(col_name.as_str()) == time_col {
            continue;
        }
        if row_idx < values.len() {
            key_parts.push(values[row_idx].clone());
        }
    }

    key_parts.sort();
    key_parts.join("|")
}
```

These functions allocate strings repeatedly per row and per indicator.

---

## Problem Statement

Actuals-heavy and sequential-heavy models can pay the same key-building cost repeatedly.

Main issues:

```text
1. dimension keys are rebuilt per row per actuals-enabled indicator
2. entity keys are rebuilt per row per indicator/function
3. key_parts Vec is allocated repeatedly
4. string format/join/sort happens repeatedly
5. forecast_indices membership uses Vec::contains in ActualsContext
6. actual_values and forecast_indices are cloned into ActualsContext
7. existing_columns lookup still uses linear scan in calculation handler
```

This can become expensive for:

```text
large row-count blocks
many actuals-enabled indicators
sequential functions like balance/change/prior
models with many dimensions
models with many forecast/actual periods
```

---

## Proposed Change

Introduce reusable per-block actuals/key metadata.

Example:

```rust
struct BlockRowKeyCache {
    row_dimension_keys: Arc<[String]>,
    row_entity_keys: Arc<[String]>,
    forecast_mask: Arc<[bool]>,
    forecast_indices: Arc<[usize]>,
}
```

For faster structured representation:

```rust
struct BlockRowKeyCache {
    row_dimension_key_ids: Arc<[u64]>,
    row_entity_key_ids: Arc<[u64]>,
    forecast_mask: Arc<[bool]>,
    forecast_indices: Arc<[usize]>,
}
```

Recommended first version:

```text
Use Arc<[String]> for compatibility.
Later, Issue 21 can replace string keys with structured IDs.
```

---

## Proposed Implementation Plan

### Phase 1: Precompute Forecast Mask Per Block

Instead of checking:

```rust
forecast_indices.contains(&row_idx)
```

use:

```rust
forecast_mask[row_idx]
```

Example:

```rust
struct ForecastInfo {
    forecast_indices: Arc<[usize]>,
    forecast_mask: Arc<[bool]>,
}
```

Then:

```rust
pub fn is_forecast(&self, row_idx: usize) -> bool {
    self.forecast_mask.get(row_idx).copied().unwrap_or(false)
}
```

---

### Phase 2: Precompute Row Dimension Keys

For each block, compute once:

```rust
row_dimension_keys[row_idx]
```

using the same logic as `build_dimension_key`.

Then `load_actuals_for_indicator` can use:

```rust
let dim_key = &row_key_cache.row_dimension_keys[row_idx];
```

instead of rebuilding the key.

---

### Phase 3: Precompute Row Entity Keys

For sequential functions and last-actual extraction, compute once:

```rust
row_entity_keys[row_idx]
```

excluding the time dimension.

Then:

```rust
let entity_key = &row_key_cache.row_entity_keys[row_idx];
```

---

### Phase 4: Avoid ActualsContext Clone Where Possible

Current code passes:

```rust
ActualsContext::new(
    actual_values.clone(),
    forecast_indices.clone(),
    last_actual_by_entity,
)
```

Change to shared data:

```rust
pub struct ActualsContext {
    pub actual_values: Arc<[f64]>,
    pub forecast_indices: Arc<[usize]>,
    pub forecast_mask: Arc<[bool]>,
    pub last_actual_by_entity: Arc<HashMap<String, f64>>,
    pub has_actuals: bool,
}
```

or use borrowed references where possible.

---

### Phase 5: Optimize Actual Map Keys

Current actual map uses string keys:

```rust
HashMap<String, f64>
```

Short-term:

```text
Reuse precomputed row_dimension_keys.
```

Long-term with Issue 21:

```text
Use structured key IDs instead of string keys.
```

---

### Phase 6: Avoid Linear Existing Column Lookup

Current pattern in calculation handler:

```rust
let existing_values = existing_columns.iter()
    .find(|(name, _)| name == node_id)
    .map(|(_, vals)| vals.clone());
```

This should be replaced through Issue 7 column store:

```rust
let existing_values = existing_column_store.get(node_id);
```

and should avoid cloning where possible.

---

## Preloaded Data Requirement

Actuals data is already part of the `CalcPlan` / `IndicatorSpec`.

This issue should not introduce Python callbacks.

Actuals handling should use:

```text
Rust-owned CalcPlan data
Rust-owned block dimension state
precomputed Rust row-key cache
ExecutionSnapshot
```

It must not use:

```rust
Python<'_>
PyObject
metadata_cache.call_method1(...)
```

---

## Expected Impact

Expected improvements:

```text
1. faster actuals loading
2. faster sequential function setup
3. faster last-actual-by-entity extraction
4. less repeated string allocation
5. less repeated sorting/joining of key parts
6. faster forecast membership checks
7. reduced memory churn
```

Practical impact:

```text
Models without actuals: low
Actuals-heavy models: high
Sequential-heavy models: high
Large row-count models: medium/high
Wide models with many actuals indicators: high
```

---

## Risks / Edge Cases

```text
1. Must preserve exact key semantics.
2. Dimension key ordering must match current sorted behavior.
3. Entity key must still exclude time dimension.
4. Forecast start date comparison must remain identical.
5. Existing actuals behavior must match Python parity.
6. Memory usage may increase slightly due to cached keys.
7. If row keys are cached as strings, memory may increase but CPU should decrease.
8. Structured key migration should be separate from first version.
```

---

## Dependencies

Recommended dependencies:

```text
Issue 7 - Optimize CalcObjectState column storage and lookup
Issue 8 - Reduce cloning and reuse preloaded/shared data
Issue 21 - Structured IDs / non-string internal keys, for long-term optimization
```

Can be implemented independently with string-based row key cache first.

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

# ISSUE 21

## Title

Introduce Structured Node, Block, Dimension, Property, and Column IDs to Reduce String Parsing and Lookup Cost

## Context / Problem

Current execution encodes semantic identity into strings:

```text
ind123
_456
prop789
b123
block123___ind456
dim123___prop456
```

This requires repeated parsing and string formatting across executor, resolver, formula handling, joins, dependency planning, and future graph construction.

Existing wrappers are not sufficient:

```rust
pub struct BlockId(pub String);
pub struct IndicatorId(pub String);
pub struct ColumnId(pub String);
```

These wrappers improve type labeling but still store strings and do not represent semantic numeric IDs such as `DimensionId(i64)` or `ColumnId::Property { ... }`.

## Goal

Introduce semantic typed IDs and conversion helpers so hot execution paths can avoid repeated string parsing while external output remains unchanged.

This is a gradual migration issue, not a big-bang rewrite.

## Current Gap In Code

Current code still uses:

```text
String node IDs
String block keys
String column names
String property references
String cross-object references
String join paths
HashMap<String, ...>
```

The resolver uses `NodeMapKey` with string fields. Join alignment uses string join paths. Formula AST column references are still strings. `PreloadedMetadata` uses numeric IDs internally, but execution often converts those IDs back into strings.

## Required Changes

### 1. Add Semantic ID Types

Add numeric typed wrappers:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct BlockId(pub i64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct IndicatorId(pub i64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct DimensionId(pub i64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct PropertyId(pub i64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct NodeId(pub i64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct ScenarioId(pub i64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct ItemId(pub i64);
```

### 2. Add Structured Column Identity

Add:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum ColumnId {
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

Keep external string conversion:

```rust
to_external_name()
parse_external_name(...)
```

### 3. Centralize Parsing Helpers

Add helpers for current formats:

```rust
parse_block_key("b123") -> BlockId
parse_indicator_column("ind123") -> IndicatorId
parse_dimension_column("_123") -> DimensionId
parse_property_column("dim123___prop456") -> (DimensionId, PropertyId)
parse_cross_object_ref("block123___ind456") -> (BlockId, IndicatorId)
```

Do not leave ad-hoc `strip_prefix`, `split`, and `format!` logic scattered across hot paths.

### 4. Migrate ColumnStore To Support Typed IDs

After string-keyed `ColumnStore` is stable, support:

```rust
ColumnStore<ColumnId, T>
```

or maintain both:

```text
internal ColumnId index
external name mapping for RecordBatch output
```

### 5. Migrate Resolver Planning Gradually

Start with resolver/node map identity:

```rust
struct TypedNodeMapKey {
    source_block_id: BlockId,
    target_block_id: BlockId,
    variable: ColumnId,
}
```

This should replace repeated string parsing in cross-object reference resolution over time.

### 6. Align With PreloadedMetadata

`PreloadedMetadata` already uses numeric IDs:

```rust
HashMap<i64, Vec<DimensionItem>>
HashMap<(i64, i64, i64), HashMap<i64, String>>
```

Typed IDs should let execution look up preloaded metadata without converting numeric IDs into strings and back again.

### 7. Preserve External Compatibility

Do not change:

```text
RecordBatch field names
Python result shape
formula syntax
warning/error text readability
API output names
```

## Dependencies

```text
Should be implemented after Source Issue 7.
Should follow Source Issue 13 for narrow hot-path interning unless typed IDs are needed sooner.
Works better once ColumnStore and shared storage are stable.
Related to future scheduler graph and formula dependency planning.
```

## Risks / Tradeoffs

```text
1. Broad migration can be risky if done all at once.
2. Formula parser still produces string column names initially.
3. External APIs require string names, so conversion boundaries must be explicit.
4. Existing tests may assert exact string names.
5. Typed IDs and interned IDs must have clear responsibilities.
6. Join paths may require a separate row-key optimization later.
```

## Acceptance Criteria

```text
1. Semantic typed IDs exist for block, indicator, dimension, property, node, scenario, item, and column identity.
2. Parsing helpers centralize existing string parsing behavior.
3. External name conversion helpers preserve current output names exactly.
4. At least one internal path uses typed IDs instead of ad-hoc string parsing.
5. ColumnStore has a documented path to typed ColumnId lookup.
6. Resolver/node-map lookup has a documented or partial typed-ID migration.
7. PreloadedMetadata lookup can use typed numeric IDs directly in new code.
8. Debug/error/warning messages remain readable.
9. Existing serial outputs remain unchanged.
10. Tests cover parse/to_external_name roundtrips for all supported formats.
11. No Python/PyO3 callbacks are introduced.
```

# Recommended Priority for Issues 19-21

```text
1. Issue 19 - Optimize FormulaEvaluator Context Cloning and Shared Column Access
2. Issue 20 - Optimize Actuals Handling, Forecast Indices, and Entity Key Reuse
3. Issue 21 - Introduce Structured Node and Column Identifiers
```

Reason:

```text
Issue 19 is likely to provide the most direct runtime benefit because FormulaEvaluator is a hot path.

Issue 20 is important for actuals-heavy and sequential-heavy models.

Issue 21 is a broader long-term data-structure cleanup that should be done gradually after the graph/column-store architecture is stable.
```

---

# Final Coverage Note

With Issues 1-21, the Omni-Calc performance roadmap covers:

```text
safe scheduler architecture
parallel ready-node execution
Rayon configuration
preloaded-only execution
no Python callbacks in Rust hot path
better column storage
less cloning
shared references
join-key optimization
formula evaluator clone reduction
actuals/entity-key optimization
typed IDs
RecordBatch materialization
connected dimension preload
join alignment
lookup aggregation
benchmarking and regression gates
```

The key rule remains:

```text
Use preloaded Rust-owned data during execution.
Do not call Python/PyO3 from Omni-Calc execution hot path after preload.
Avoid unnecessary clones.
Use shared references / Arc where safe.
Use better data structures before adding more parallelism.
```
