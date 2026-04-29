# Jira Ticket Set: Omni-Calc Rust Parallelism Opportunities After Preload Snapshot Refactor

---

## Ticket 1

### Issue Type:
Task / Tech Debt

### Title:
Refactor Omni-Calc executor around immutable execution snapshots and per-node outputs

### Summary:
Refactor the Rust Omni-Calc executor so node execution no longer mutates the global `ExecutionContext` directly. This is the prerequisite for true parallel execution inside Rust now that required metadata is increasingly preloaded into Rust-owned structures.

### Background / Context:
In the current branch, `modelAPI/omni-calc/src/engine/exec/executor.rs` still processes Python-provided `calc_steps` sequentially.

Current flow:

```rust
for step in &plan.request.calc_steps {
    match step.calc_type.as_str() {
        "input" => process_input_step(&mut ctx, step),
        "calculation" => process_calculation_step(&mut ctx, step),
        "sequential" => process_sequential_step(&mut ctx, step),
        _ => {}
    }
}
```

The executor comments explicitly state that `calc_steps` are already ordered by the Python DAG manager and Rust simply processes them in order.

The shared mutable execution state is stored in `ExecutionContext` in:

```text
modelAPI/omni-calc/src/engine/exec/context.rs
```

`ExecutionContext` currently owns:

```rust
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
```

This design is safe for serial execution but not structured for safe concurrent node execution.

### Problem Statement:
Most executor functions take `&mut ExecutionContext`.

This means each node execution can directly mutate:

- `ctx.calc_object_states`
- `ctx.resolver`
- `ctx.warnings`
- `ctx.nodes_calculated`
- `ctx.string_property_map_cache`
- `ctx.numeric_property_map_cache`

Using `Arc<Mutex<ExecutionContext>>` around the entire executor would technically be thread-safe, but it would serialize most work behind one global lock and would not provide meaningful parallel speedup.

Bad pattern:

```rust
let mut ctx = ctx.lock().unwrap();
process_entire_node(&mut ctx, node);
```

This keeps expensive formula evaluation, dependency resolution, column construction, resolver update, warning collection, and state mutation inside one lock.

### Scope:
Refactor executor internals to separate:

1. read-only node input,
2. node calculation,
3. synchronized output merge.

This ticket should not introduce full parallel execution yet. It should prepare the execution model for parallelism.

### Technical Analysis:
Current mutation points in `process_calculation_step` include:

```rust
ctx.warnings.push(warning);
state.number_columns.push((col_name, values));
state.connected_dim_columns.push((col_name, values));
ctx.nodes_calculated += step_res.count;
ctx.update_resolver(&block_key);
```

Current calculation execution mixes:

- dependency resolution,
- property loading,
- formula evaluation,
- state mutation,
- resolver materialization,
- warning collection.

This makes it difficult to safely run multiple ready nodes concurrently.

Because the branch now preloads metadata into Rust-owned `PreloadedMetadata`, a worker can theoretically calculate a node without Python callbacks if all required inputs are captured in an immutable snapshot.

### Proposed Change:
Introduce an internal read-only snapshot type, for example:

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
}
```

Introduce an output type:

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

Refactor node execution from:

```rust
fn process_node(ctx: &mut ExecutionContext, node_id: &str)
```

to:

```rust
fn execute_node(snapshot: ExecutionSnapshot, node: ExecNode) -> NodeOutput
```

Then centralize mutation:

```rust
fn merge_node_output(ctx: &mut ExecutionContext, output: NodeOutput) {
    if let Some(state) = ctx.calc_object_states.get_mut(&output.block_key) {
        for col in output.number_columns {
            if !state.number_columns.iter().any(|(name, _)| name == &col.0) {
                state.number_columns.push(col);
            }
        }

        for col in output.string_columns {
            state.string_columns.push(col);
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

### Parallelism Opportunity:
Level: task scheduling / worker orchestration, independent calc execution, dependency graph layer execution.

This is the foundational refactor that enables real parallelism later.

Once node execution returns `NodeOutput` and reads only from immutable snapshots, worker threads can execute independent nodes without holding a global mutable lock.

### Expected Impact:
- Enables safe parallel execution in later tickets.
- Reduces need for global locking.
- Makes executor behavior easier to reason about.
- Centralizes mutation and makes deterministic output ordering possible.
- Makes race conditions easier to prevent.

### Risks / Edge Cases:
- Must preserve exact current output behavior.
- Column insertion order may affect downstream behavior if any logic assumes order.
- Duplicate column handling must remain deterministic.
- Resolver update timing must not break cross-object references.
- Warnings must preserve stable ordering where tests or users depend on it.

### Dependencies:
None.

This is the first prerequisite ticket.

### Acceptance Criteria:
- Node execution logic can return a `NodeOutput` without directly mutating `ExecutionContext`.
- `merge_node_output` or equivalent centralized merge function exists.
- Existing serial execution path still produces the same results.
- Existing tests pass.
- Sequential steps still execute atomically.
- Cross-object references still resolve correctly.
- No Python callbacks are introduced during node execution.

### Testing Notes:
Add tests for:

- merging numeric columns,
- merging string columns,
- merging connected dimension columns,
- duplicate column avoidance,
- warning merge behavior,
- node count merge behavior,
- resolver update trigger behavior.

### Out of Scope:
- Rayon integration.
- Full Kahn scheduler.
- Work-stealing.
- Parallelizing sequential groups.
- Changing calculation semantics.

### Priority:
Highest

### Labels:
`omni-calc`, `rust`, `parallelism`, `executor`, `tech-debt`, `kahn-scheduler`, `preload`

### Components:
Omni-Calc, Rust Engine, Calculation Executor

---

## Ticket 2

### Issue Type:
Task / Tech Debt

### Title:
Build explicit Rust-side execution graph from CalcPlan for Kahn-style scheduling

### Summary:
Introduce a Rust-side execution graph that models calculation nodes, dependencies, outgoing edges, and in-degree counts. This will replace the current implicit dependency model where Rust simply trusts Python-ordered `calc_steps`.

### Background / Context:
The current graph-like data is represented in:

```text
modelAPI/omni-calc/src/engine/integration/calc_plan.rs
```

Relevant structures:

```rust
pub struct CalcPlan {
    pub blocks: HashMap<String, BlockSpec>,
    pub dimensions: HashMap<String, DimensionSpec>,
    pub calc_steps: Vec<CalcStep>,
    pub node_maps: Vec<PlannedNodeMap>,
    pub variable_filters: HashMap<String, VariableFilter>,
    pub property_specs: HashMap<String, PropertySpec>,
}
```

```rust
pub struct CalcStep {
    pub calc_type: String,
    pub nodes: Vec<String>,
}
```

`calc_steps` currently provide implicit ordering. Rust does not currently maintain:

- explicit node graph,
- in-degree counts,
- outgoing adjacency list,
- ready queue,
- dependency-completion tracking.

### Problem Statement:
The current Rust executor cannot safely determine which nodes are ready to run independently.

It knows the order of steps, but does not explicitly know:

- which nodes depend on which input indicators,
- which nodes depend on cross-object references,
- which nodes depend on dimension properties,
- which nodes are sequential groups,
- which nodes are safe to run in parallel.

Without an explicit graph, Rust cannot implement Kahn-style parallel scheduling correctly.

### Scope:
Add an internal graph representation that can be built from existing `CalcPlan` data while preserving current behavior.

### Technical Analysis:
The graph can be constructed using:

- `CalcPlan.calc_steps`
- `CalcPlan.blocks`
- `BlockSpec.indicators`
- `IndicatorSpec.parsed_formula`
- `CalcPlan.property_specs`
- `CalcPlan.node_maps`
- `CalcPlan.variable_filters`

Cross-object dependencies are especially important because `CrossObjectResolver` expects source block data to already exist before resolving a reference like:

```text
block39951___ind259068
```

Sequential steps should be represented as grouped atomic nodes because `process_sequential_step` explicitly processes all dependencies first and then processes the group together.

### Proposed Change:
Introduce internal graph types:

```rust
struct ExecutionGraph {
    nodes: HashMap<String, ExecNode>,
    outgoing: HashMap<String, Vec<String>>,
    indegree: HashMap<String, usize>,
}
```

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

```rust
enum ExecNodeType {
    InputIndicator,
    Property,
    Calculation,
    SequentialGroup,
}
```

For sequential steps, create one graph node per sequential group:

```rust
ExecNodeType::SequentialGroup
```

with:

```rust
outputs = step.nodes
parallel_safe = false
```

### Parallelism Opportunity:
Level: dependency graph layer execution, task scheduling / worker orchestration.

This enables Kahn-style scheduling:

```text
ready queue -> execute ready nodes -> merge outputs -> decrement dependents -> enqueue newly ready nodes
```

### Expected Impact:
- Makes dependencies explicit.
- Enables correctness checks before parallelism.
- Enables future ready-node batching.
- Enables detection of independent nodes.
- Enables better debugging of scheduling behavior.
- Allows Rust to validate Python DAG order instead of blindly trusting it.

### Risks / Edge Cases:
- Formula parsing must correctly detect intra-block and cross-block dependencies.
- Cross-object references may appear in parsed formula names and must map to source block and source node.
- Property nodes may have dependency relationships not currently explicit.
- Sequential group dependency boundaries must be preserved.
- Existing Python DAG ordering must remain compatible during migration.

### Dependencies:
- Ticket 1 is recommended before actual parallel execution.
- This ticket can be implemented before or alongside Ticket 1 if it is initially analysis-only and does not change execution behavior.

### Acceptance Criteria:
- Rust can build an `ExecutionGraph` from `CalcPlan`.
- Every `CalcStep` node is represented.
- Sequential steps are represented as atomic grouped nodes.
- Cross-object dependencies from formulas / `PlannedNodeMap` are represented.
- Graph can produce a deterministic topological order.
- Single-threaded graph order matches current `calc_steps` behavior for existing test plans.
- Cycle detection exists and returns useful diagnostics.
- Debug output can show node id, block key, dependencies, and dependents.

### Testing Notes:
Add tests for:

- simple input -> calculation dependency,
- multiple independent input nodes,
- cross-block dependency through `blockX___indY`,
- property dependency,
- sequential group atomic node,
- cycle detection,
- graph order matching current `calc_steps`.

### Out of Scope:
- Parallel execution.
- Work-stealing.
- Changing Python DAG planner.
- Removing `calc_steps`.

### Priority:
Highest

### Labels:
`omni-calc`, `rust`, `scheduler`, `dag`, `kahn-scheduler`, `dependency-graph`

### Components:
Omni-Calc, Rust Engine, Planner, Executor

---

## Ticket 3

### Issue Type:
Story

### Title:
Implement single-threaded Kahn scheduler before enabling parallel execution

### Summary:
Replace the direct `for calc_step in calc_steps` executor loop with a single-threaded Kahn scheduler over the new Rust-side `ExecutionGraph`.

### Background / Context:
Current execution is:

```rust
for step in &plan.request.calc_steps {
    match step.calc_type.as_str() {
        "input" => process_input_step(&mut ctx, step),
        "calculation" => process_calculation_step(&mut ctx, step),
        "sequential" => process_sequential_step(&mut ctx, step),
        _ => {}
    }
}
```

This gives serial execution based on Python-provided ordering.

Before adding parallelism, Rust should first prove it can execute the same plan using an explicit ready queue and dependency tracking.

### Problem Statement:
Jumping directly from ordered `calc_steps` to parallel workers risks changing semantics and making bugs hard to isolate.

A single-threaded Kahn scheduler gives a safe intermediate step:

```text
ready queue
execute one ready node
merge output
decrement dependents
enqueue newly ready nodes
```

### Scope:
Implement graph-driven execution in one thread.

### Technical Analysis:
This should use the graph from Ticket 2 and the snapshot/output/merge model from Ticket 1.

Pseudo-flow:

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

    for dep in graph.outgoing[&node_id].iter() {
        graph.indegree[dep] -= 1;
        if graph.indegree[dep] == 0 {
            ready.push_back(dep.clone());
        }
    }
}
```

### Proposed Change:
Add a new execution path, initially behind a feature flag or config toggle:

```rust
execute_with_kahn_scheduler(ctx, graph)
```

Keep the existing execution path available until parity is proven.

### Parallelism Opportunity:
Level: dependency graph layer execution.

This ticket does not introduce parallelism yet, but it creates the exact scheduling model required for parallel ready-node execution.

### Expected Impact:
- Proves Rust-side dependency tracking.
- Provides a safer migration path.
- Helps identify hidden dependencies that were previously only enforced by Python ordering.
- Makes future parallel execution much lower risk.

### Risks / Edge Cases:
- Current `calc_steps` may include ordering assumptions not captured in parsed formulas.
- Some dependencies may be implicit through resolver materialization timing.
- Sequential groups must not be split.
- Resolver update timing must match current semantics.
- Warning ordering may change unless explicitly preserved.

### Dependencies:
- Ticket 1.
- Ticket 2.

### Acceptance Criteria:
- Single-threaded Kahn execution produces the same output as current serial execution.
- Existing tests pass under both old and new execution paths.
- Sequential groups remain atomic.
- Cross-object references resolve correctly.
- Resolver snapshots are updated at correct dependency boundaries.
- Useful diagnostics are returned if graph execution cannot proceed due to missing dependencies or cycles.

### Testing Notes:
Run output parity tests on:

- pure input plans,
- simple calculated indicators,
- cross-block references,
- property joins,
- filtered references,
- sequential formulas,
- actuals / forecast-start logic.

### Out of Scope:
- Rayon.
- Parallel worker execution.
- Work-stealing.
- Optimization of resolver materialization.

### Priority:
High

### Labels:
`omni-calc`, `rust`, `kahn-scheduler`, `single-threaded`, `dag`

### Components:
Omni-Calc, Rust Engine, Executor

---

## Ticket 4

### Issue Type:
Story

### Title:
Parallelize independent input indicator execution within ready graph layers

### Summary:
Once the executor uses immutable snapshots and `NodeOutput`, parallelize independent input indicator nodes that do not depend on each other and only produce distinct output columns.

### Background / Context:
Input nodes are currently processed by:

```text
modelAPI/omni-calc/src/engine/exec/executor.rs
```

through:

```rust
fn process_input_step(ctx: &mut ExecutionContext, step: &CalcStep) {
    for node_id in &step.nodes {
        if node_id.starts_with("prop") {
            process_property_node(ctx, node_id);
        } else if node_id.starts_with("ind") {
            process_input_indicator_node(ctx, node_id);
        }
    }
}
```

Input value loading is handled by:

```text
modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs
```

The input handler parses input JSON, builds dimension mappings, applies forecast start filtering, handles actuals, and returns `StepResult`.

### Problem Statement:
Input indicator nodes are processed sequentially even when they are independent.

For example, multiple input indicators in the same ready layer can be loaded independently because each produces its own `ind{id}` column.

### Scope:
Parallelize only independent input indicator nodes after graph/snapshot/output refactor is complete.

### Technical Analysis:
`InputStepHandler::load_input_values` is mostly local computation:

- parse `data_values_json`,
- create `DimensionMapper`,
- create `TimeUtils`,
- load based on input type,
- apply actuals / forecast filter.

The result is a vector for a single indicator column.

Because metadata is already present in the `CalcPlan` and input data is embedded in `BlockSpec.input_data`, input indicator processing does not require Python callbacks.

### Proposed Change:
After Kahn scheduler identifies a batch/layer of ready input indicator nodes, execute them with Rayon:

```rust
use rayon::prelude::*;

let outputs: Vec<NodeOutput> = ready_input_nodes
    .into_par_iter()
    .map(|node| {
        let snapshot = build_snapshot_readonly(&ctx, &node);
        execute_input_node(snapshot, node)
    })
    .collect();

for output in outputs {
    merge_node_output(&mut ctx, output);
}
```

Use Rayon because:

- CPU-bound work,
- no async IO,
- no Python callback needed,
- `rayon` is already present in `Cargo.toml`.

### Parallelism Opportunity:
Level: independent calc execution, dependency graph layer execution.

### Expected Impact:
- Speeds up models with many input indicators.
- Reduces startup calculation time for wide models.
- Safe first parallel execution target because input nodes have simple output ownership.

### Risks / Edge Cases:
- Same indicator must not be emitted twice.
- Warning ordering must be deterministic.
- Output merge ordering should be stable.
- Input JSON parsing overhead may become memory-pressure heavy if many large inputs are parsed concurrently.
- `BlockSpec` and required input data should be shared read-only via `Arc`.

### Dependencies:
- Ticket 1.
- Ticket 2.
- Ticket 3.

### Acceptance Criteria:
- Independent input nodes in the same ready layer can execute in parallel.
- Each worker returns `NodeOutput`.
- No worker mutates `ExecutionContext`.
- Merge order is deterministic.
- Results match serial execution.
- Parallel path can be disabled via config / feature flag.
- Sequential and calculation nodes are not accidentally parallelized by this ticket.

### Testing Notes:
Add tests for:

- multiple independent input indicators in one block,
- input indicators across multiple blocks,
- actuals merge behavior,
- forecast start filtering,
- raw / constant / growth input types,
- deterministic output comparison between serial and parallel paths.

### Out of Scope:
- Parallel property loading.
- Parallel formula evaluation.
- Parallel cross-object resolution.
- Sequential groups.

### Priority:
High

### Labels:
`omni-calc`, `rust`, `rayon`, `parallel-inputs`, `performance`

### Components:
Omni-Calc, Rust Engine, Input Execution

---

## Ticket 5

### Issue Type:
Story

### Title:
Parallelize preloaded property map population and property node execution

### Summary:
Use the new Rust-side `PreloadedMetadata` path to parallelize property map creation and property node execution without Python callbacks.

### Background / Context:
This branch adds:

```text
modelAPI/omni-calc/src/engine/exec/preload.rs
```

with:

```rust
pub struct PreloadedMetadata {
    pub dimension_items: HashMap<i64, Vec<DimensionItem>>,
    pub property_maps: HashMap<(i64, i64, i64), HashMap<i64, String>>,
}
```

Property loading now has snapshot-based Rust functions in:

```text
modelAPI/omni-calc/src/engine/exec/get_source_data/dim_loader.rs
```

including:

```rust
load_string_property_map_from_snapshot(...)
load_property_map_from_snapshot(...)
```

Input property processing in:

```text
modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs
```

now calls the snapshot-based loaders through `process_properties`.

### Problem Statement:
Property nodes and property-map cache population are still sequential and rely on mutable caches in `ExecutionContext`:

```rust
pub string_property_map_cache: HashMap<(i64, i64, i64), StringPropertyMap>,
pub numeric_property_map_cache: HashMap<(i64, i64, i64), (PropertyMap, Vec<String>)>,
```

`process_properties` takes:

```rust
string_cache: &mut HashMap<...>
numeric_cache: &mut HashMap<...>
```

This prevents safe parallel property processing.

### Scope:
Refactor property map cache population to happen before parallel execution, or make it read-only during worker execution.

### Technical Analysis:
Because property data is now available in `PreloadedMetadata`, property map creation no longer needs Python callbacks if the required properties are preloaded.

Current branch still contains old Python-callback loaders:

```rust
load_property_map(py, metadata_cache, ...)
load_string_property_map(py, metadata_cache, ...)
```

but the newer snapshot functions avoid Python:

```rust
load_string_property_map_from_snapshot(snapshot, property_spec, scenario_id)
load_property_map_from_snapshot(snapshot, property_spec, scenario_id)
```

This creates a valid opportunity to precompute all property maps from `plan.property_specs` before node execution begins.

### Proposed Change:
Add a pre-execution property cache build phase:

```rust
struct PropertyCacheSnapshot {
    string_maps: HashMap<(i64, i64, i64), Arc<StringPropertyMap>>,
    numeric_maps: HashMap<(i64, i64, i64), Arc<(PropertyMap, Vec<String>)>>,
}
```

Build it from:

```rust
plan.request.property_specs
ctx.preloaded_metadata
plan.request.scenario_id
```

Potential Rayon usage:

```rust
let maps: Vec<_> = property_specs
    .par_iter()
    .map(|(node_id, spec)| build_property_cache_entry(snapshot, spec, scenario_id))
    .collect();
```

Then workers read from immutable `Arc<PropertyCacheSnapshot>`.

### Parallelism Opportunity:
Level: cache population, preload/deserialization work, independent calc execution.

### Expected Impact:
- Removes mutable cache bottleneck from property execution.
- Makes property node execution safe for parallel workers.
- Avoids repeated property-map construction.
- Avoids Python GIL entirely during property node execution.

### Risks / Edge Cases:
- Must preserve missing-value behavior, including `NaN` for missing numeric property values.
- Must preserve warning behavior from `parse_property_value`.
- Must handle linked-dimension properties differently from text and numeric properties.
- Must avoid duplicate work for repeated property specs.
- Memory usage may increase if all property maps are precomputed.

### Dependencies:
- Ticket 1.
- Ticket 2 recommended.
- Requires complete preload coverage for all `property_specs`.

### Acceptance Criteria:
- Property maps can be built from `PreloadedMetadata` without Python callbacks.
- Runtime property node execution does not mutate shared property caches.
- Property node execution can return `NodeOutput`.
- Results match current serial behavior.
- Missing metadata produces clear errors or warnings.
- Old Python callback loaders are not used during Rust executor execution when preload is available.

### Testing Notes:
Add tests for:

- numeric property map from snapshot,
- text property map from snapshot,
- linked-dimension property map from snapshot,
- missing property values,
- parse warnings,
- cache reuse,
- serial-vs-parallel property output parity.

### Out of Scope:
- Removing old Python fallback functions entirely.
- Parallel formula evaluation.
- Parallel resolver updates.

### Priority:
High

### Labels:
`omni-calc`, `rust`, `preload`, `property-cache`, `rayon`, `parallelism`

### Components:
Omni-Calc, Rust Engine, Metadata Preload, Property Loading

---

## Ticket 6

### Issue Type:
Story

### Title:
Parallelize independent calculation nodes within graph ready layers

### Summary:
After the executor has explicit graph scheduling and per-node outputs, run independent calculation nodes in the same ready layer concurrently.

### Background / Context:
Current calculation step processing is in:

```text
modelAPI/omni-calc/src/engine/exec/executor.rs
```

`process_calculation_step` loops over each calculation node:

```rust
for node_id in &step.nodes {
    ...
}
```

For each node, it:

1. finds the block,
2. clones dimension and existing numeric columns,
3. collects connected dims needed for cross-object joins,
4. resolves cross-object dependencies,
5. collects dimension property columns,
6. collects connected dimension columns,
7. mutates block state,
8. evaluates formula,
9. pushes results,
10. updates resolver.

### Problem Statement:
Even when two calculation nodes are independent and ready, the executor processes them sequentially.

This misses a major opportunity for true Rust-side parallelism now that metadata can be preloaded and formula evaluation can run without Python callbacks.

### Scope:
Parallelize only calculation nodes that are proven independent by the Rust-side graph.

### Technical Analysis:
Formula evaluation itself is Rust-local through:

```text
modelAPI/omni-calc/src/engine/exec/formula_eval.rs
```

`FormulaEvaluator` owns an evaluation context and evaluates formulas against column vectors.

A calculation node can run in parallel if:

- all dependencies are already merged,
- required cross-object columns are available in its snapshot,
- required property columns are available from immutable property cache,
- the node writes unique output columns,
- the node is not part of a sequential group,
- the node does not rely on another node in the same ready batch.

### Proposed Change:
Use Rayon for CPU-bound parallel calculation nodes:

```rust
let outputs: Vec<NodeOutput> = ready_calc_nodes
    .into_par_iter()
    .map(|node| {
        let snapshot = build_snapshot_readonly(&ctx, &node);
        execute_calculation_node(snapshot, node)
    })
    .collect();

for output in outputs {
    merge_node_output(&mut ctx, output);
}
```

Execution should not call `ctx.update_resolver` inside workers. Resolver update should happen during merge or after the layer.

### Parallelism Opportunity:
Level: independent calc execution, dependency graph layer execution, batch evaluation.

### Expected Impact:
- Speeds up wide calculation layers.
- Benefits models with many independent calculated indicators.
- Moves CPU-bound formula work across available cores.

### Risks / Edge Cases:
- Hidden dependencies in formulas must be correctly detected.
- Column availability must be snapshot-consistent.
- Warning ordering must be deterministic.
- Resolver snapshots must not be mutated by workers.
- Existing same-step intra-evaluator behavior must not be assumed for independent nodes.
- Some formulas may have sequential/time-series behavior and should be excluded unless proven safe.

### Dependencies:
- Ticket 1.
- Ticket 2.
- Ticket 3.
- Ticket 5 recommended for property dependencies.

### Acceptance Criteria:
- Independent calculation nodes can run in parallel.
- Dependent calculation nodes do not run until dependencies are merged.
- Workers do not mutate `ExecutionContext`.
- Cross-object references resolve from read-only resolver snapshot.
- Results match serial execution.
- Parallelism can be disabled by config / feature flag.
- Sequential groups are excluded.

### Testing Notes:
Add tests for:

- two independent formulas in same block,
- independent formulas across blocks,
- dependent formulas in same block,
- cross-object dependency ordering,
- property-dependent formulas,
- warning determinism,
- serial-vs-parallel parity.

### Out of Scope:
- Work-stealing scheduler.
- Parallel sequential groups.
- Changing formula semantics.
- Optimizing expression evaluator internals.

### Priority:
High

### Labels:
`omni-calc`, `rust`, `formula-eval`, `rayon`, `parallel-calc`, `performance`

### Components:
Omni-Calc, Rust Engine, Formula Evaluation, Executor

---

## Ticket 7

### Issue Type:
Task / Performance

### Title:
Reduce resolver materialization frequency and make resolver snapshots read-only

### Summary:
Refactor resolver updates so `ctx.update_resolver(&block_key)` is not called after every individual calculation node. Create stable read-only resolver snapshots for worker execution and update resolver only at dependency-safe boundaries.

### Background / Context:
`ExecutionContext::update_resolver` is defined in:

```text
modelAPI/omni-calc/src/engine/exec/context.rs
```

Current implementation:

```rust
pub fn update_resolver(&mut self, block_key: &str) {
    if let Some(state) = self.calc_object_states.get(block_key) {
        if let Ok(batch) = build_record_batch(state) {
            self.resolver.add_block(block_key.to_string(), batch);
        }
    }
}
```

`build_record_batch` clones columns into Arrow arrays.

### Problem Statement:
`process_calculation_step` calls:

```rust
ctx.update_resolver(&block_key);
```

after each node.

This means a block `RecordBatch` can be rebuilt repeatedly:

```text
node 1 -> rebuild full block RecordBatch
node 2 -> rebuild full block RecordBatch
node 3 -> rebuild full block RecordBatch
```

This is expensive and blocks parallel execution because the resolver is a shared mutable structure.

### Scope:
Refactor resolver updates to happen at controlled merge boundaries.

### Technical Analysis:
`CrossObjectResolver` in:

```text
modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs
```

stores:

```rust
calculated_blocks: HashMap<String, BlockData>
```

and `resolve_reference` requires source block data to already exist.

This is correct for serial execution, but for parallel execution workers should read from a stable resolver snapshot, not mutate the shared resolver.

### Proposed Change:
Introduce a read-only resolver snapshot:

```rust
struct CrossObjectResolverSnapshot {
    calculated_blocks: Arc<HashMap<String, BlockData>>,
    node_maps: Arc<HashMap<NodeMapKey, PlannedNodeMap>>,
    variable_filters: Arc<HashMap<String, VariableFilter>>,
}
```

Resolver update policy options:

1. Update after each graph layer.
2. Update only when a downstream dependency requires the new block output.
3. Update after merge of nodes that produce cross-object source columns.

Recommended first implementation:

```text
single-threaded Kahn merge
-> merge outputs for node/layer
-> update resolver for affected block once
-> release dependents
```

### Parallelism Opportunity:
Level: snapshot/materialization work, task scheduling / worker orchestration, IO boundary removal.

### Expected Impact:
- Reduces repeated `RecordBatch` rebuilds.
- Reduces shared mutable resolver contention.
- Makes cross-object reads safe for worker threads.
- Improves performance even before full parallelism.

### Risks / Edge Cases:
- Resolver update timing is semantically important.
- Downstream nodes must not see partially merged block state.
- Cross-object references may require connected dimension columns as well as numeric indicator columns.
- If updated too late, dependent nodes may fail to resolve source columns.
- If updated too early, workers may see inconsistent state.

### Dependencies:
- Ticket 1.
- Ticket 2.
- Ticket 3 recommended.

### Acceptance Criteria:
- Resolver update frequency is reduced from per-node where safe.
- Resolver snapshots are read-only during worker execution.
- Cross-object references still resolve correctly.
- Existing serial behavior remains unchanged.
- Tests cover source block availability and indicator-column availability.
- No worker directly mutates `ctx.resolver`.

### Testing Notes:
Add tests for:

- cross-block reference after source node merge,
- multiple source nodes in same block,
- connected dimension columns in resolver batch,
- resolver update after graph layer,
- missing source block diagnostics,
- serial-vs-new resolver parity.

### Out of Scope:
- Rewriting `CrossObjectResolver` join logic.
- Full parallel formula execution.
- Removing `RecordBatch` output format.

### Priority:
High

### Labels:
`omni-calc`, `rust`, `resolver`, `recordbatch`, `parallelism`, `performance`

### Components:
Omni-Calc, Rust Engine, Cross-Object Resolver

---

## Ticket 8

### Issue Type:
Performance

### Title:
Parallelize final block RecordBatch materialization

### Summary:
Build final output `RecordBatch` values for blocks in parallel after execution completes.

### Background / Context:
Final result building is currently done in:

```text
modelAPI/omni-calc/src/engine/exec/context.rs
```

The function:

```rust
fn build_execution_result(ctx: &ExecutionContext, start: Instant) -> Result<CalcResult>
```

loops through `ctx.calc_object_states`, filters block states, and calls:

```rust
build_record_batch(state)
```

for each block.

`build_record_batch` clones:

- `dim_columns`,
- `connected_dim_columns`,
- `string_columns`,
- `number_columns`

into Arrow arrays.

### Problem Statement:
Final materialization is independent per block but currently sequential.

If a model has many blocks or large block states, building Arrow `RecordBatch` outputs can be CPU/memory-copy heavy.

### Scope:
Parallelize final block `RecordBatch` construction across blocks.

### Technical Analysis:
At the end of execution, no more mutation should be happening. `CalcObjectState` can be read immutably.

This makes final materialization a relatively safe parallelization target.

The work is CPU/memory-copy bound, not IO-bound, and does not require Python.

### Proposed Change:
Use Rayon to parallelize block batch construction:

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

Then merge into `CalcResult` deterministically, optionally sorting by block key:

```rust
let mut batches = batches;
batches.sort_by(|a, b| a.0.cmp(&b.0));

for (block_key, batch) in batches {
    result.add_block(block_key, batch);
}
```

### Parallelism Opportunity:
Level: snapshot/materialization work.

### Expected Impact:
- Quick win after ensuring `CalcObjectState` is read-only at result build time.
- Speeds up large models with many blocks.
- No dependency graph changes required.

### Risks / Edge Cases:
- Must preserve deterministic block order if consumers depend on it.
- Memory pressure may increase because multiple large Arrow arrays are built at once.
- Should cap Rayon parallelism or allow config if memory spikes.

### Dependencies:
None strictly.

Can be implemented independently, but safer after Ticket 1 if state ownership is cleaned up.

### Acceptance Criteria:
- Final `RecordBatch` creation can run in parallel per block.
- Output result matches serial implementation.
- Block output ordering is deterministic.
- Errors during batch creation are handled consistently with current behavior.
- Parallel path is covered by tests.

### Testing Notes:
Add tests for:

- multiple block outputs,
- string columns,
- connected dimension columns,
- numeric columns,
- duplicate column warning behavior,
- serial-vs-parallel result parity.

### Out of Scope:
- Changing `RecordBatch` schema.
- Changing result format.
- Parallelizing per-column materialization inside one block.

### Priority:
Medium

### Labels:
`omni-calc`, `rust`, `recordbatch`, `rayon`, `quick-win`, `performance`

### Components:
Omni-Calc, Rust Engine, Result Materialization

---

## Ticket 9

### Issue Type:
Performance / Tech Debt

### Title:
Parallelize connected dimension preload across blocks

### Summary:
Refactor `preload_connected_dimensions` so connected dimension columns can be computed per block in parallel and then merged into block states.

### Background / Context:
Connected dimension preload is in:

```text
modelAPI/omni-calc/src/engine/exec/executor.rs
```

Function:

```rust
fn preload_connected_dimensions(ctx: &mut ExecutionContext)
```

It loops through every block in the plan, checks each dimension, reads `property_values`, builds connected dimension columns, and then mutates the corresponding `CalcObjectState`.

### Problem Statement:
Connected dimension preload is sequential across blocks.

The computation for each block is mostly independent:

- read block spec,
- read dimension specs,
- read existing block state dimensions,
- build connected dimension columns,
- return columns to add.

The only shared mutation is appending the resulting connected columns to `ctx.calc_object_states`.

### Scope:
Split connected dimension preload into:

1. parallel read-only compute phase,
2. deterministic merge phase.

### Technical Analysis:
Current function mixes computation and mutation:

```rust
for (block_key, block_spec) in &ctx.plan.request.blocks {
    ...
    connected_dims_to_add.push((col_name, connected_values));
    ...
    if let Some(state) = ctx.calc_object_states.get_mut(block_key) {
        for (col_name, values) in connected_dims_to_add {
            state.connected_dim_columns.push((col_name, values));
        }
    }
}
```

This can be refactored into:

```rust
fn compute_connected_dims_for_block(snapshot, block_key, block_spec) -> BlockConnectedDimOutput
```

and then:

```rust
fn merge_connected_dims(ctx, outputs)
```

### Proposed Change:
Use Rayon across blocks:

```rust
let outputs: Vec<BlockConnectedDimOutput> = block_keys
    .par_iter()
    .map(|block_key| compute_connected_dims_for_block(...))
    .collect();

for output in outputs {
    merge_connected_dims(&mut ctx, output);
}
```

### Parallelism Opportunity:
Level: preload/deserialization work, cache population, independent block preprocessing.

### Expected Impact:
- Speeds up connected dimension preprocessing for models with many blocks/dimensions.
- Moves one more pre-execution bottleneck into parallel Rust.
- Keeps merge small and deterministic.

### Risks / Edge Cases:
- Existing state is read while outputs are computed; must avoid mutable borrow during parallel compute.
- Duplicate connected columns must still be skipped.
- Merge order should be deterministic.
- Heavy logging inside preload may become noisy/interleaved if done in parallel; consider reducing or structuring logs.

### Dependencies:
Ticket 1 recommended, but this can be done independently with a local compute-output-merge refactor.

### Acceptance Criteria:
- Connected dimension columns are computed without mutating `ExecutionContext`.
- Per-block connected dimension computation can run in parallel.
- Merge preserves current behavior.
- Output matches serial preload behavior.
- Duplicate column skipping remains correct.
- Logging remains usable.

### Testing Notes:
Add tests for:

- block with no connected dimensions,
- block with connected dimension property values,
- multiple blocks with connected dimensions,
- duplicate connected dimension columns,
- serial-vs-parallel preload parity.

### Out of Scope:
- Changing connected dimension semantics.
- Changing Python payload format.
- Parallelizing formula execution.

### Priority:
Medium

### Labels:
`omni-calc`, `rust`, `preload`, `connected-dimensions`, `rayon`, `performance`

### Components:
Omni-Calc, Rust Engine, Preload

---

## Ticket 10

### Issue Type:
Performance / Tech Debt

### Title:
Parallelize join-path creation and target alignment for large cross-object joins

### Summary:
Optimize `node_alignment` join path creation and alignment for large row counts by introducing parallel implementations where beneficial.

### Background / Context:
Cross-object alignment uses:

```text
modelAPI/omni-calc/src/engine/exec/node_alignment/join_path.rs
modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs
modelAPI/omni-calc/src/engine/exec/node_alignment/mod.rs
```

Current path building:

```rust
pub fn build_all_join_paths(...) -> Vec<String> {
    (0..row_count)
        .map(|row_idx| build_join_path(dim_columns, dim_names, row_idx))
        .collect()
}
```

Current target alignment:

```rust
for (row_idx, path) in target_join_paths.iter().enumerate() {
    let value = lookup.get(path).copied().unwrap_or(default_value);
    result.push(value);
}
```

### Problem Statement:
For large source/target row counts, join path construction and target alignment are CPU/memory intensive and currently sequential.

### Scope:
Add parallel variants for large datasets only.

### Technical Analysis:
The following operations are naturally parallel per row:

- source join path construction,
- target join path construction,
- target alignment from read-only lookup map.

However, lookup map creation with aggregation is more complex because it mutates a `HashMap`.

Safe first target:

- parallelize `build_all_join_paths`,
- parallelize `align_with_lookup`.

Do not initially parallelize `create_lookup_map` aggregation unless using a correct reduce strategy.

### Proposed Change:
Add threshold-based parallel functions:

```rust
pub fn build_all_join_paths_parallel(...)
pub fn align_with_lookup_parallel(...)
```

Use sequential implementation below a row threshold to avoid overhead.

Potential implementation:

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

### Parallelism Opportunity:
Level: aggregation/reduction, batch evaluation, cross-object alignment.

### Expected Impact:
- Improves performance for large cross-block joins.
- Useful for models with high row counts and many cross-object references.
- Does not require full scheduler parallelism.

### Risks / Edge Cases:
- String allocation is heavy; parallelizing may increase memory pressure.
- `HashMap` read access is safe if immutable, but ensure type is `Sync`.
- Debug counters like `missing_count` and sums in `align_with_lookup` are currently mutable and sequential; parallel version needs either reduction or simplified stats.
- Small datasets may become slower due to Rayon overhead.
- Deterministic result ordering must be preserved.

### Dependencies:
None strictly.

Ticket 7 is recommended to reduce resolver materialization overhead first.

### Acceptance Criteria:
- Parallel path is used only above configurable row threshold.
- Output ordering matches serial implementation.
- Results match serial implementation.
- Missing/default behavior matches serial implementation.
- Parallel stats/logging does not introduce race conditions.
- Benchmarks show improvement for large row counts.

### Testing Notes:
Add tests for:

- exact output parity with serial join paths,
- missing target keys,
- no common dimensions,
- aggregation + alignment,
- large synthetic row count benchmark,
- threshold fallback behavior.

### Out of Scope:
- Parallelizing `create_lookup_map` aggregation in first version.
- Replacing string join paths with encoded row keys.
- Changing join semantics.

### Priority:
Medium

### Labels:
`omni-calc`, `rust`, `node-alignment`, `join-path`, `rayon`, `performance`

### Components:
Omni-Calc, Rust Engine, Node Alignment

---

## Ticket 11

### Issue Type:
Performance / Research

### Title:
Evaluate parallel aggregation for lookup map creation with deterministic reductions

### Summary:
Investigate and implement parallel lookup-map aggregation for large cross-object joins only if deterministic and faster than current sequential `HashMap` aggregation.

### Background / Context:
Lookup map creation is in:

```text
modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs
```

Current aggregation:

```rust
for (path, &value) in join_paths.iter().zip(values.iter()) {
    *lookup.entry(path.clone()).or_insert(0.0) += value;
}
```

Other modes include:

- `mean`,
- `first`,
- `last`,
- default `sum`.

### Problem Statement:
For large source row counts, lookup map creation can be expensive.

However, naïvely parallelizing writes into a shared `HashMap` would require locking and may be slower or nondeterministic.

### Scope:
Evaluate parallel aggregation using per-thread local maps and deterministic reduction.

### Technical Analysis:
Safe approach:

```text
split rows into chunks
each worker builds local HashMap
merge local maps deterministically
```

For `sum` and `mean`, reduction is straightforward.

For `first` and `last`, deterministic behavior depends on original row order:

- `first` must preserve earliest row,
- `last` must preserve latest row.

That requires tracking row index alongside value.

### Proposed Change:
Add optional parallel aggregation for large datasets:

```rust
enum AggValue {
    Sum(f64),
    Mean { sum: f64, count: usize },
    First { index: usize, value: f64 },
    Last { index: usize, value: f64 },
}
```

Use per-thread maps and reduce.

Use a threshold to avoid parallel overhead on small datasets.

### Parallelism Opportunity:
Level: aggregation/reduction.

### Expected Impact:
- May improve performance for large grouped cross-object references.
- Complements Ticket 10.
- Reduces cost of `CrossObjectResolver::apply_node_map`.

### Risks / Edge Cases:
- Floating-point summation order may change results slightly.
- `first` / `last` must be deterministic by source row index.
- More memory usage due to per-thread maps.
- May not outperform sequential version for small/medium datasets.
- Requires benchmarks before enabling by default.

### Dependencies:
Ticket 10 recommended first.

### Acceptance Criteria:
- Parallel aggregation is deterministic.
- Aggregation modes match serial behavior.
- Floating-point differences are documented and within acceptable tolerance.
- Threshold-based fallback exists.
- Benchmarks justify enabling it.

### Testing Notes:
Add tests for:

- sum aggregation,
- mean aggregation,
- first aggregation,
- last aggregation,
- duplicate paths,
- deterministic ordering,
- large row benchmark.

### Out of Scope:
- Changing aggregation semantics.
- Replacing string keys with interned keys.
- Full resolver rewrite.

### Priority:
Low / Medium

### Labels:
`omni-calc`, `rust`, `aggregation`, `node-alignment`, `rayon`, `performance-research`

### Components:
Omni-Calc, Rust Engine, Node Alignment

---

## Ticket 12

### Issue Type:
Task

### Title:
Remove or isolate remaining Python callback paths from Rust execution hot path

### Summary:
Audit and isolate remaining PyO3 metadata callback functions so Rust worker execution paths use only `PreloadedMetadata`.

### Background / Context:
This branch added `PreloadedMetadata` and snapshot-based property loading.

However, old Python callback loaders still exist in:

```text
modelAPI/omni-calc/src/engine/exec/get_source_data/dim_loader.rs
```

Examples:

```rust
pub fn load_property_map(py: Python, metadata_cache: &PyObject, ...)
pub fn load_string_property_map(py: Python, metadata_cache: &PyObject, ...)
```

The new preferred paths are:

```rust
load_string_property_map_from_snapshot(...)
load_property_map_from_snapshot(...)
```

The Python boundary in:

```text
modelAPI/omni-calc/src/python.rs
```

does:

```rust
let preloaded = preload_metadata(py, metadata_cache.as_ref(), &plan.inner)?;
engine.set_preloaded_metadata(preloaded);

let result = py.allow_threads(|| runtime::execute(&mut engine, plan.inner.clone()))?;
```

So Rust execution already runs outside the GIL after preload.

### Problem Statement:
To safely add Rust worker threads, worker execution paths must not call Python or require GIL.

Remaining callback functions should either be:

- removed from execution paths,
- isolated behind non-parallel fallback,
- or guarded so they cannot run in parallel mode.

### Scope:
Audit all Rust execution code paths and ensure parallel-eligible paths use only Rust-owned preload data.

### Technical Analysis:
Parallel worker execution cannot safely rely on:

```rust
Python<'_>
PyObject
metadata_cache.call_method1(...)
metadata_cache.getattr(...)
```

because this would reintroduce:

- GIL contention,
- Python-side shared mutable cache concerns,
- blocking behavior,
- non-Rust scheduling constraints.

### Proposed Change:
Add explicit execution mode validation:

```rust
enum MetadataAccessMode {
    PreloadedOnly,
    PythonFallbackAllowed,
}
```

Parallel execution should require:

```rust
MetadataAccessMode::PreloadedOnly
```

If required metadata is missing from `PreloadedMetadata`, return a clear error before worker execution starts.

### Parallelism Opportunity:
Level: IO boundary removal, Python FFI boundary removal.

### Expected Impact:
- Makes Rust parallel execution safe.
- Prevents accidental GIL calls inside workers.
- Makes preload completeness testable.
- Improves reliability of parallel mode.

### Risks / Edge Cases:
- Some legacy paths may still depend on Python fallback.
- Missing preload data must produce actionable errors.
- Backward compatibility may require keeping fallback for serial mode.
- Complete preload validation may expose existing plan gaps.

### Dependencies:
Ticket 5 recommended.

### Acceptance Criteria:
- Parallel-eligible execution paths do not require `Python<'_>` or `PyObject`.
- Missing preloaded metadata is detected before parallel execution.
- Old callback functions are not used in parallel mode.
- Serial fallback mode remains available if required.
- Tests verify no Python callback is needed during Rust execution after preload.

### Testing Notes:
Add tests for:

- complete preload success,
- missing dimension items,
- missing property map,
- serial fallback behavior,
- parallel mode rejecting missing preload,
- no GIL-required code in worker paths.

### Out of Scope:
- Removing Python bindings.
- Removing metadata cache class.
- Changing Python plan generation.

### Priority:
High

### Labels:
`omni-calc`, `rust`, `python-ffi`, `gil`, `preload`, `parallelism`

### Components:
Omni-Calc, Rust Engine, Python Boundary

---

## Ticket 13

### Issue Type:
Performance / Tech Debt

### Title:
Add configurable Rayon thread-pool strategy for Omni-Calc parallel execution

### Summary:
Add configuration for Rayon-based parallel execution so Omni-Calc can control worker count, disable parallel mode, and avoid oversubscription with Python/web-server concurrency.

### Background / Context:
`modelAPI/omni-calc/Cargo.toml` already includes:

```toml
rayon = "1.10"
```

But current source search did not show active Rayon usage.

### Problem Statement:
Adding Rayon directly with the global Rayon pool can cause oversubscription if the Python service is already handling multiple requests concurrently.

Parallel execution should be configurable.

### Scope:
Add engine configuration for parallelism.

Potential config:

```rust
pub struct EngineConfig {
    pub enable_parallel_execution: bool,
    pub parallel_threads: Option<usize>,
    pub parallel_row_threshold: usize,
    pub parallel_node_threshold: usize,
}
```

### Technical Analysis:
Rayon is appropriate for:

- CPU-bound formula evaluation,
- independent node execution,
- join path building,
- final materialization,
- property cache build.

Async is not appropriate for these Rust CPU-bound tasks because there is no IO wait inside the worker execution path after preload.

Scoped threads may be useful if borrowed data lifetimes make `Arc` conversion too invasive, but Rayon is preferred for work distribution once snapshots are `Send + Sync`.

Work-stealing is not recommended initially because correctness and deterministic scheduling matter more than maximum throughput.

### Proposed Change:
Add config-driven parallel strategy:

```rust
if config.enable_parallel_execution {
    execute_parallel(...)
} else {
    execute_serial(...)
}
```

Add optional custom Rayon pool:

```rust
rayon::ThreadPoolBuilder::new()
    .num_threads(n)
    .build()
```

Avoid nested parallelism where possible.

### Parallelism Opportunity:
Level: task scheduling / worker orchestration.

### Expected Impact:
- Safer deployment of parallel mode.
- Prevents uncontrolled CPU usage.
- Allows benchmarking and gradual rollout.
- Enables easy fallback to serial mode.

### Risks / Edge Cases:
- Global Rayon pool configuration can only be initialized once.
- Per-request custom pools have overhead.
- Nested Rayon calls can cause oversubscription.
- Configuration needs to align with Python service concurrency.

### Dependencies:
Should be implemented before enabling production parallel execution.

### Acceptance Criteria:
- Parallel execution can be enabled/disabled.
- Worker thread count can be configured.
- Serial fallback remains default until parity is proven.
- Benchmarks can run serial vs parallel.
- No nested parallelism surprises in common execution paths.

### Testing Notes:
Add tests for:

- serial config,
- parallel config,
- custom thread count,
- deterministic output under different thread counts,
- fallback behavior.

### Out of Scope:
- Work-stealing scheduler design.
- Async runtime integration.
- Python-level concurrency management.

### Priority:
Medium

### Labels:
`omni-calc`, `rust`, `rayon`, `config`, `performance`, `parallelism`

### Components:
Omni-Calc, Rust Engine, Runtime Config

---

## Ticket 14

### Issue Type:
Rejected / Not Recommended

### Title:
Do not use a single global Arc<Mutex<ExecutionContext>> for parallel execution

### Summary:
Avoid implementing parallel execution by wrapping the whole `ExecutionContext` in one `Arc<Mutex<_>>`.

### Background / Context:
The current executor mutates shared state through `&mut ExecutionContext`.

A simple approach would be:

```rust
let ctx = Arc::new(Mutex::new(ctx));

ready_nodes.into_par_iter().for_each(|node| {
    let mut ctx = ctx.lock().unwrap();
    process_node(&mut ctx, node);
});
```

### Problem Statement:
This makes execution thread-safe but not truly parallel.

Only one worker can hold the lock at a time, so expensive calculation work remains serialized.

### Scope:
This is a rejected implementation approach.

### Technical Analysis:
The global mutex would cover:

- dependency resolution,
- formula evaluation,
- property loading,
- state mutation,
- resolver update,
- warning collection.

This creates high lock contention and likely worse performance than serial execution.

### Proposed Change:
Do not implement this approach.

Use instead:

```text
ExecutionSnapshot = read-only worker input
NodeOutput = worker result
merge_node_output = short synchronized mutation
```

### Parallelism Opportunity:
None.

### Expected Impact:
Avoids a misleading implementation that appears parallel but performs serially.

### Risks / Edge Cases:
Using global mutex may still be useful for a temporary correctness prototype, but it should not be merged as the production design.

### Dependencies:
None.

### Acceptance Criteria:
- No production parallel execution path locks the full `ExecutionContext` during node execution.
- Any temporary prototype using this approach is clearly feature-gated and not enabled by default.

### Testing Notes:
N/A.

### Out of Scope:
N/A.

### Priority:
High as a design constraint

### Labels:
`omni-calc`, `rust`, `parallelism`, `rejected`, `mutex`, `design-constraint`

### Components:
Omni-Calc, Rust Engine, Executor

---

## Ticket 15

### Issue Type:
Rejected / Not Recommended

### Title:
Do not parallelize sequential groups initially

### Summary:
Do not split or parallelize `sequential` calc steps during the first parallel execution rollout.

### Background / Context:
`process_sequential_step` in:

```text
modelAPI/omni-calc/src/engine/exec/executor.rs
```

has a different flow from normal calculation steps:

```text
1. Process ALL dependencies for ALL nodes first
2. Then process the entire group together
```

Sequential formulas include concepts like:

- `rollfwd`
- `prior`
- `balance`
- `change`

These depend on time ordering and entity history.

### Problem Statement:
Sequential groups have internal state and ordering semantics. Splitting them into parallel node execution could break correctness.

### Scope:
Treat each sequential step as one atomic scheduler node.

### Technical Analysis:
Sequential execution uses prior-period and entity-level state. Even if metadata is preloaded, the computation itself is not independent per node in the same way as regular formulas.

### Proposed Change:
Represent sequential steps as:

```rust
ExecNodeType::SequentialGroup
```

with:

```rust
parallel_safe = false
```

The group may be scheduled after dependencies are ready, but the internal execution should remain serial for now.

### Parallelism Opportunity:
Rejected for now.

Potential future research could parallelize across independent entity groups inside sequential evaluation, but that is a separate, higher-risk optimization.

### Expected Impact:
Preserves correctness during initial parallel rollout.

### Risks / Edge Cases:
None from rejecting this optimization.

### Dependencies:
None.

### Acceptance Criteria:
- Sequential groups are not split into independent parallel nodes.
- Kahn scheduler treats each sequential step as atomic.
- Existing sequential tests continue to pass.
- Future TODO can mention possible entity-level parallelization research.

### Testing Notes:
Add tests proving sequential group atomic execution order is preserved.

### Out of Scope:
- Entity-level sequential parallelism.
- Rewriting sequential evaluator.

### Priority:
High as a design constraint

### Labels:
`omni-calc`, `rust`, `sequential`, `parallelism`, `rejected`, `correctness`

### Components:
Omni-Calc, Rust Engine, Sequential Evaluation

---

# Prioritized Recommended Tickets

1. Ticket 1 — Refactor executor around immutable snapshots and per-node outputs
2. Ticket 2 — Build explicit Rust-side execution graph
3. Ticket 3 — Implement single-threaded Kahn scheduler
4. Ticket 12 — Remove/isolate remaining Python callback paths from Rust execution hot path
5. Ticket 5 — Parallelize preloaded property map population and property node execution
6. Ticket 7 — Reduce resolver materialization frequency and make resolver snapshots read-only
7. Ticket 4 — Parallelize independent input indicator execution
8. Ticket 6 — Parallelize independent calculation nodes
9. Ticket 13 — Add configurable Rayon thread-pool strategy
10. Ticket 8 — Parallelize final block RecordBatch materialization
11. Ticket 9 — Parallelize connected dimension preload across blocks
12. Ticket 10 — Parallelize join-path creation and target alignment
13. Ticket 11 — Evaluate parallel aggregation for lookup map creation
14. Ticket 14 — Rejected: global `Arc<Mutex<ExecutionContext>>`
15. Ticket 15 — Rejected: parallelizing sequential groups initially

---

# Dependency Map

```text
Ticket 1
  -> Ticket 3
  -> Ticket 4
  -> Ticket 5
  -> Ticket 6
  -> Ticket 7

Ticket 2
  -> Ticket 3
  -> Ticket 4
  -> Ticket 6

Ticket 3
  -> Ticket 4
  -> Ticket 6

Ticket 5
  -> Ticket 6
  -> Ticket 12

Ticket 7
  -> Ticket 6
  -> Ticket 10

Ticket 13
  -> should be completed before production rollout of Ticket 4 / Ticket 6 / Ticket 10

Ticket 8
  -> can be done independently

Ticket 9
  -> can be done independently after compute/merge split

Ticket 10
  -> can be done independently, but benefits from Ticket 7

Ticket 11
  -> depends on Ticket 10 or should follow Ticket 10
```

---

# Quick Wins vs Larger Refactors

## Quick Wins

### Ticket 8 — Parallelize final block RecordBatch materialization
Why quick:
- End-of-execution state is read-only.
- Work is naturally per-block.
- Low dependency on scheduler refactor.

### Ticket 9 — Parallelize connected dimension preload across blocks
Why quick-ish:
- Per-block computation is mostly independent.
- Needs compute/merge split but not full graph scheduler.

### Ticket 10 — Parallelize join-path creation and target alignment
Why quick-ish:
- Row-level map operations are pure/read-only.
- Needs thresholding to avoid overhead.

---

## Larger Refactors

### Ticket 1 — Immutable snapshots and NodeOutput
Why larger:
- Requires restructuring executor internals.
- Must preserve all current semantics.

### Ticket 2 — Rust-side execution graph
Why larger:
- Requires dependency extraction from formulas, node maps, properties, and calc steps.

### Ticket 3 — Single-threaded Kahn scheduler
Why larger:
- Changes core scheduling model.
- Needs strong parity testing.

### Ticket 6 — Parallel calculation nodes
Why larger:
- Requires graph correctness, snapshot correctness, resolver correctness, and deterministic merge.

### Ticket 7 — Resolver snapshot and materialization strategy
Why larger:
- Cross-object references are semantically sensitive.
- Incorrect resolver timing can break calculations.

---

# Recommended First Implementation Path

```text
1. Implement Ticket 1.
2. Implement Ticket 2.
3. Implement Ticket 3 and prove serial parity.
4. Implement Ticket 12 to ensure no Python callbacks in worker paths.
5. Implement Ticket 5 to make property cache read-only.
6. Implement Ticket 7 to make resolver snapshots safe.
7. Implement Ticket 4 as first real parallel node execution.
8. Implement Ticket 6 after input/property parallelism is proven.
```

Final note:

The branch’s preload work creates real parallelism opportunities because property metadata can now be loaded into Rust-owned `PreloadedMetadata`, and the Rust executor is already invoked under `py.allow_threads`.

However, true parallelism should not be added by locking the whole `ExecutionContext`.

The correct path is:

```text
immutable snapshots
+ per-node outputs
+ deterministic merge
+ explicit Rust-side DAG scheduling
+ preloaded-only worker execution
```
