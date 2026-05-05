# Rewritten Omni-Calc Source Issues: Column Storage, Shared Data, Clone Reduction, Formula Context, Interning, and Typed IDs

Source of truth reviewed:

```text
/Users/veerpratapsingh/Downloads/omni-calc-tasks-improvements-main/29-april-omni-calc-parallelisation-FINAL.md
```

Validated against current Rust codebase:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc
```

Recommended implementation order:

```text
1. Source Issue 7  - ColumnStore and indexed lookup
2. Source Issue 11 - Snapshot-friendly Arc-backed shared column storage
3. Source Issue 8  - Clone reduction using shared columns and preloaded data
4. Source Issue 19 - FormulaEvaluator shared context and clone reduction
5. Source Issue 13 - String interning for selected hot-path keys
6. Source Issue 21 - Structured typed IDs and gradual migration
```

---

# Source Issue 7 - Add ColumnStore and Indexed Lookup

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

# Source Issue 11 - Add Snapshot-Friendly Shared Arc Column Storage

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

# Source Issue 8 - Reduce Cloning Using Shared Columns and Preloaded Data

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

# Source Issue 19 - Refactor FormulaEvaluator Shared Context

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

# Source Issue 13 - Add String Interning for Selected Hot-Path Keys

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

# Source Issue 21 - Add Structured Typed IDs and Gradual Migration

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
