# 16. Relations

A `Relation` is a declarative logical-plan node. Its `rel_type` oneof selects one operator; `RelationCommon` carries optional plan identity and origin metadata. Constructing or analyzing a relation MUST NOT execute command side effects.

## 16.1 Common relation contract

Every relation MUST select exactly one variant and provide each required child. Except for the two client-only invalid shapes below, a server MUST reject an invalid enum, contradictory alternatives, a missing required child or field, an unsupported variant, or a cycle through the Chapter 7 envelope. It MUST NOT discard an unknown wrapper and execute its child.\
Within the Chapter 16 relation surface, v1.0 defines exactly two client-only invalid-shape exceptions: an `AsOfJoin` that sets both `join_expr` and `using_columns`, and a `StatFreqItems` request with an empty `cols` list. A conforming client MUST NOT emit either shape. Server behavior for either shape is outside v1.0 conformance, and the core TCK MUST NOT assert a server outcome for it. No other invalid relation inherits this exception. Sections 17.1 and 19.1 independently govern invalid expression and command shapes; §22.6 requires those shapes to be included in the Required Profile source-validation audit.\
`RelationCommon.plan_id` is a client plan identifier used for references, metrics, and diagnostics. It is not a globally accessible server object ID. `origin` is diagnostic and MUST NOT affect plan results.\
Portable results include column order, names, qualifiers, types, nullability, metadata, row multiplicity, and documented ordering. Physical plans, task counts, and partition boundaries are not normative unless a public operator exposes them.

## 16.2 v1.0 required relation profile

`SC-1.0-P1-WIRE` requires every standard portable Spark 4.2.0 batch relation below. A canonical constraint may narrow a field domain; omission from a client API does not remove the wire requirement.

| Group | Required relation variants |
| :---- | :---- |
| Sources and roots | `Read` with `is_streaming=false`, `LocalRelation`, `Range`, and `SQL` as a dialect-declared transport |
| Projection and selection | `Project`, `Filter`, `WithColumns`, `WithColumnsRenamed`, `Drop`, `ToDF`, `ToSchema` |
| Combination | `Join`, `AsOfJoin`, `LateralJoin`, `NearestByJoin` with the required exact-mode row, `SetOperation`, `WithRelations` |
| Aggregation and observation | `Aggregate`, `CollectMetrics` |
| Ordering and row selection | `Sort`, `Limit`, `Offset`, `Tail`, `Sample`, `Deduplicate` with `within_watermark=false` |
| Distribution and naming | `SubqueryAlias`, `Repartition`, `RepartitionByExpression`, `Hint` |
| Shape and parsing | `Unpivot`, `Transpose`, `Parse` |
| DataFrame helpers | `NAFill`, `NADrop`, `NAReplace`, `StatSummary`, `StatCrosstab`, `StatDescribe`, `StatCov`, `StatCorr`, `StatApproxQuantile`, `StatFreqItems`, `StatSampleBy` |
| Catalog | The `Catalog` operations marked Required in Chapter 12 |

The following variants are outside the core profile:

* worker/UDF nodes: `MapPartitions`, `GroupMap`, `CoGroupMap`, `CommonInlineUserDefinedTableFunction`, and `CommonInlineUserDefinedDataSource`;\
* streaming nodes and modes: `WithWatermark`, `ApplyInPandasWithState`, streaming reads, and watermark deduplication;\
* `MlRelation`;\
* dialect-owned `UnresolvedTableValuedFunction`;\
* vendor-specific `RelationChanges`;\
* presentation nodes `ShowString` and `HtmlString`;\
* cache transport nodes `CachedLocalRelation`, `ChunkedCachedLocalRelation`, and `CachedRemoteRelation`; and\
* `extension` and `Unknown`.

An implementation may support an excluded variant, but it does not count toward core conformance.

## 16.3 Sources

### 16.3.1 `Read`

`Read` selects either `NamedTable` or `DataSource`. A named-table identifier is parsed as a multipart identifier; a server MUST NOT concatenate it into SQL text. Data-source provider names, schemas, paths, options, defaults, and errors follow Appendix H. Provider and option names are case-insensitive where Appendix H says so.\
The core profile requires only batch reads. A server MUST NOT run `is_streaming=true` as a finite batch and report success.

### 16.3.2 `LocalRelation`

`LocalRelation.data`, when present, is Arrow IPC streaming data. An optional supplied schema must agree with the data. If data is absent, a schema is required and represents an empty relation. Invalid or incompatible Arrow data is an analysis error, not permission for arbitrary coercion.

### 16.3.3 `Range`

`Range.end` and `step` are required; `start` defaults to zero. Step zero and non-positive partition counts are invalid. The result is one `Long` column and follows the pinned endpoint and overflow rows.

## 16.4 Projection, filtering, and column shape

`Project` may omit its input for a constant projection but requires at least one expression. Output order equals expression order. `Filter` requires a Boolean-compatible condition; it retains only rows for which the condition is true.\
`WithColumns` adds or replaces columns in request order. `WithColumnsRenamed` applies ordered renames. `Drop` supports its expression and name forms. `ToDF` requires one name per input column. `ToSchema` validates the supplied structured schema and applies the canonical coercion rules.\
Missing, duplicate, or ambiguous names follow the structured analysis-error rows. A server MUST NOT choose an arbitrary match.

## 16.5 Joins and relation composition

### 16.5.1 Ordinary joins

`Join` requires left and right inputs and a non-unspecified type. A condition and `using_columns` are mutually exclusive. Inner, cross, left/right/full outer, left-semi, and left-anti forms preserve row multiplicity and the documented output schema. A `using_columns` join emits one copy of each using column and resolves names under the required case-sensitivity configuration. A condition that evaluates to NULL does not match. Outer joins introduce NULL only on the non-preserved side. The relation does not promise row order.

### 16.5.2 `AsOfJoin`

`AsOfJoin` requires both inputs and both as-of expressions. The core accepts `join_type` values `inner` and `leftouter`, and `direction` values `backward`, `forward`, and `nearest`. A valid core request sets at most one of `join_expr` and `using_columns`; the selected form restricts eligible right rows in addition to the as-of predicate. A conforming client MUST NOT send both; §16.1 classifies that invalid shape. NULL or false join predicates and NULL as-of values are not eligible.\
For a left value `l` and right value `r`, eligibility is:

* `backward`: `r <= l`, or `r < l` when `allow_exact_matches=false`;\
* `forward`: `r >= l`, or `r > l` when `allow_exact_matches=false`; and\
* `nearest`: either side of `l`, excluding `r = l` when `allow_exact_matches=false`.

A tolerance, when present, MUST be a foldable, non-negative value compatible with the as-of difference. The distance bound is inclusive when `allow_exact_matches=true` and strict when it is false. Among eligible rows, backward chooses the smallest `l-r`, forward chooses the smallest `r-l`, and nearest chooses the smallest absolute difference. Exactly one right row is selected per matched left row. When several right rows have the same minimum distance, the selected right row is unspecified; a conforming test accepts any tied candidate and MUST NOT infer a physical input order. `inner` drops an unmatched left row. `leftouter` emits it once with NULL right columns. Output order is unspecified.

### 16.5.3 `NearestByJoin`

For each left row, `NearestByJoin` ranks every right row whose ranking expression is non-NULL. `num_results` is from 1 through 100000, `join_type` is `inner` or `leftouter`, and `direction` is `distance` or `similarity`. The Required Profile uses `mode=exact`; `mode=approx` remains optional until a separate profile defines a recall or error bound. Exact mode requires an orderable, deterministic ranking expression.\
The operator selects at most `num_results` candidates. `distance` selects the smallest ranking values; `similarity` selects the largest. Equal ranking values have no secondary key. If a tie crosses the `num_results` boundary, any subset of the tied rows that fills the remaining positions is valid, and order within a tie is unspecified. This matches the pinned API contract, which does not define tie-breaking. The relation itself is unordered; clients that require presentation order add `Sort`.\
An `inner` nearest join drops a left row when no non-NULL-ranked candidate exists. A `leftouter` nearest join emits that left row once with NULL right columns. The TCK MUST cover unique ranks, NULL ranks, empty right input, both directions, both join types, and a boundary tie without requiring one physical tied row.

### 16.5.4 Lateral joins and relation references

`LateralJoin` evaluates the right relation with the permitted correlation to the current left row and preserves the selected inner or left-outer semantics. `WithRelations` carries a root plus named or plan-ID references. References may depend on earlier references but MUST NOT form a cycle or cross sessions. Expansion preserves the same plan semantics as the corresponding repeated tree.

## 16.6 Set operations and aggregation

### 16.6.1 Set operations

`SetOperation` requires two inputs and `UNION`, `INTERSECT`, or `EXCEPT`. `is_all` controls whether multiplicity is preserved or duplicate rows are removed. By-name and allow-missing-column modes are valid only for union. By-name union resolves names under the required case-sensitivity setting, preserves the left schema order, reorders matching right columns, and fills permitted missing columns with NULL. Incompatible arity or types fail analysis. No set operation promises row order.

### 16.6.2 Aggregate output contract

`Aggregate` requires an input, a non-unspecified `group_type`, and aggregate expressions from the required Appendix C rows. For group-by, rollup, cube, and grouping sets, the output begins with all `grouping_expressions` in request order and then the `aggregate_expressions` in request order. Aliases determine asserted output names. Rows are unordered unless a parent `Sort` orders them.\
NULL grouping keys form one group. With no grouping key, an ordinary aggregate returns one row even for empty input; each aggregate function supplies its Appendix C empty-input value. With one or more active grouping keys, empty input returns no group. Rollup, cube, and grouping sets may still produce an empty-key grouping set and therefore one aggregate row on empty input.

### 16.6.3 Grouping forms

For grouping expressions `(g1, ..., gn)`:

* `GROUP_TYPE_GROUPBY` uses the single grouping set `(g1, ..., gn)`.\
* `GROUP_TYPE_ROLLUP` uses the ordered prefixes `(g1, ..., gn)`, `(g1, ..., g(n-1))`, through `()`.\
* `GROUP_TYPE_CUBE` uses every subset of the grouping expressions.\
* `GROUP_TYPE_GROUPING_SETS` uses exactly the sets carried in `grouping_sets`; every expression in a set MUST also occur in `grouping_expressions`.

An inactive grouping expression is NULL in the output. That generated NULL is not distinguishable from an input NULL unless the profile later adds the `grouping` or `grouping_id` functions. Repeated grouping sets preserve their result multiplicity. The TCK compares row multisets rather than physical group order.\
For the input `(A,X,1)`, `(A,Y,2)`, `(B,X,3)` and aggregate `sum(v)`, required examples include: group-by `k1` gives `(A,3)` and `(B,3)`; rollup `(k1,k2)` adds `(A,NULL,3)`, `(B,NULL,3)`, and `(NULL,NULL,6)` to the three detailed groups; cube also adds `(NULL,X,4)` and `(NULL,Y,2)`; grouping sets `[(k1),(k2),()]` produce the two `k1` totals, the two `k2` totals, and the grand total. These are unordered multisets.

### 16.6.4 Pivot

`GROUP_TYPE_PIVOT` requires `pivot.col`. `grouping_expressions` identify output rows. Each pivot value, in value-list order, is crossed with each aggregate expression, in aggregate-list order, to form output columns. With one aggregate, a pivoted column name is the pivot value's string form (`null` for NULL). With multiple aggregates, the name is `<pivot-value>_<aggregate-alias-or-expression-name>`. A value is cast to the pivot-column type before null-safe equality matching; a non-foldable or incompatible value fails analysis.\
When `pivot.values` is empty, analysis collects the distinct pivot values, sorts them by the pivot-column ordering, and rejects more than `spark.sql.pivotMaxValues`. This discovery may execute the input as an analysis-time read, including during `AnalyzePlan.Schema`, but it does not execute the plan to produce result rows. It is an analysis-time read governed by §9.2. The implementation MUST materialize at most spark.sql.pivotMaxValues \+ 1 distinct values, using the extra value only to detect overflow; it MUST expose no result rows, execute no command, and mutate no session-scoped state. A required TCK case uses explicit values; separate cases verify omitted-value schema discovery, the configured limit, and the absence of result, command, and session-state side effects.\
For the example above, pivoting `k2` over explicit values `X,Y`, grouping by `k1`, and applying `sum(v)` yields columns `k1,X,Y` and unordered rows `(A,1,2)` and `(B,3,NULL)`.

## 16.7 Ordering, limits, sampling, and deduplication

`Sort.is_global=true` requests total ordering; false requests partition-local ordering. Each sort expression carries explicit direction and null ordering.\
`Limit`, `Offset`, and `Tail` require non-negative bounds. They do not add an implicit sort. Without a documented input order, row identity is nondeterministic.\
`Sample` validates bounds, replacement mode, deterministic-order flag, and seed. New clients provide a seed even though the wire field remains optional for compatibility. Equal seeds do not promise equal samples after physical partitioning changes.\
`Deduplicate` accepts either named keys or all columns, not both. The core requires batch behavior only; `within_watermark=true` belongs to Streaming.

## 16.8 Distribution, aliases, hints, and metrics

`Repartition.num_partitions` must be positive. `RepartitionByExpression` requires partition expressions and validates an optional positive partition count. `DirectShufflePartitionID` is not a core expression and cannot be used to satisfy a core row.\
`SubqueryAlias` changes resolution scope, not values. `Hint` requires a name and input. An optimizer may ignore an advisory hint, but the server must parse and validate it and must not change query results because the hint is unrecognized.\
`CollectMetrics` requires an input, metric name, and metric expressions. Core metrics use required aggregate/function rows. The response preserves metric names, types, and values under Chapter 10.

## 16.9 Unpivot, transpose, NA, and statistics

### 16.9.1 Unpivot, transpose, and NA operations

`Unpivot` preserves identifier columns in request order, followed by the requested variable and value columns. An omitted values field derives every non-identifier input column; a present empty values list is distinct and follows its canonical row. Every value column must have a common required type under the canonical cast rows. Each input row emits one output row per value column, including a row when that value is NULL.\
`Transpose` validates its optional index expression. With an index, each distinct non-NULL index value names one output column; without one, the first input column supplies the index. The output schema, string conversion used for generated names, duplicate-index rejection, NULL handling, and deterministic name ordering are fixed by its `REL-TRANSPOSE-*` rows. The TCK uses unique required-type index values and asserts those rows rather than an engine's display formatting.\
`NAFill` replaces NULL values only in selected columns compatible with the replacement's logical type. `NADrop` keeps a row exactly when the selected subset contains at least the requested minimum number of non-NULL values; an omitted subset means all columns. `NAReplace` applies each old-to-new mapping only to selected type-compatible columns. Missing, ambiguous, duplicate, or incompatible column/replacement inputs fail analysis. None of these operations treats an empty string, zero, or false as NULL.

### 16.9.2 Summary and describe

`StatSummary` accepts `count`, `mean`, `stddev`, `min`, `max`, `count_distinct`, `approx_count_distinct`, and percentile strings from `0%` through `100%`. An empty list means `count`, `mean`, `stddev`, `min`, `25%`, `50%`, `75%`, and `max`, in that order. The output begins with a non-null String column named `summary`, followed by each eligible numeric or string input column in input order, represented as nullable String values. Output rows follow the requested statistic order. Unsupported statistic names fail analysis.\
`StatDescribe` selects the named columns, or all numeric and string columns when none are named. It returns the same String-valued shape for exactly `count`, `mean`, `stddev`, `min`, and `max`, in that order. NULL input values do not contribute to count or numeric statistics. An empty eligible column set still returns the `summary` column and five statistic rows.

### 16.9.3 Crosstab, covariance, and correlation

`StatCrosstab(col1,col2)` groups NULL as the string `null`, stringifies other values, and returns counts for every observed pair. Its first String column is named `<col1>_<col2>` and contains the distinct `col1` values; one non-null Long column is produced for each distinct `col2` value under the canonical column-name and ordering rows. Missing or ambiguous columns fail analysis.\
`StatCov` and `StatCorr` require two resolvable numeric columns and return one non-null Double column named `cov` or `corr`. Before calculation, each NULL input is converted to `0.0` and every non-NULL numeric value to Double. `StatCov` computes sample covariance and returns `0.0` when it is undefined. `StatCorr` accepts only an absent method or the exact lowercase string `pearson`, computes Pearson correlation, and returns NaN when it is undefined. Any other method spelling or a nonnumeric column fails analysis.

### 16.9.4 Quantiles, frequent items, and stratified sampling

`StatApproxQuantile` preserves requested column and probability order. Each probability must be in `[0,1]`; `relative_error` must be non-negative and values above 1 are treated as 1. The result is one non-null `array<array<double>>` column named `approx_quantile`, with one inner array per requested column and one value per probability. NULL and NaN inputs are ignored; a column with no remaining value produces an empty inner array. With error `e`, probability `p`, and `N` eligible values, the returned value has rank from `floor((p-e)N)` through `ceil((p+e)N)`, clipped to the valid rank range. Error zero requires an exact quantile.\
A valid core `StatFreqItems` request names one or more columns and sets support in `[0.0001,1]`; omitted support is `0.01`. A conforming client MUST NOT send an empty `cols` list; §16.1 classifies that invalid shape. The relation returns one array column named `<input>_freqItems` per requested column. Every value whose frequency is at least the support threshold must appear; false positives are permitted by the algorithm and output order is unspecified.\
`StatSampleBy` samples without replacement by the value of its stratum expression. Each fraction must be in `[0,1]`, duplicate stratum literals are invalid, and an unlisted stratum has fraction zero. The Required Profile requires an explicit seed. For the same input multiset, stratum expression, fractions, seed, and partitioning, the selected multiset must repeat; the relation itself does not promise row order.\
The TCK covers each accepted domain and each specified rejection boundary above, NULL/NaN inputs, empty input, exact and approximate quantiles, absent strata, and deterministic seeded sampling. It compares unordered relations as multisets and applies the published approximation bounds instead of requiring bit-identical physical algorithms.

## 16.10 `Parse`

`Parse` requires a single-text-column input and a non-unspecified CSV, JSON, or XML format. An explicit schema uses required data types. Without one, the server infers the schema under the canonical format and option rows. That inference may read the input relation during analysis and is governed by §9.2: it MUST perform only schema-determination work, honor the format's sampling and inference options, expose no result rows, execute no command, and mutate no session-scoped state. Option keys are case-insensitive. Invalid input, schema, or options produce structured errors.\
The TCK covers explicit and inferred schemas for each required format, including sampling-option boundaries, empty input, malformed records under each required parse mode, and the §9.2 side-effect restrictions.

## 16.11 Catalog relation

The Catalog variant is required only for the operations listed in Chapter 12. Identifiers, result schemas, ordering, side effects, and in-scope errors follow that chapter. Authentication and authorization remain deployment-specific. The relation wrapper does not make optional Catalog operations required.

## 16.12 SQL relation

`Relation.SQL` carries query text plus named or positional expression parameters. It is a required transport node and MUST accept every `SC-1.0-P1-PORTABLE-SQL` query in Appendix I. Named and positional forms that the proto declares mutually exclusive MUST NOT coexist. Missing, extra, or invalid parameter bindings fail before execution with the Chapter 7 envelope.\
The required query result follows Chapters 10, 14, and 15. The server may parse the text natively or translate it to its own plan/dialect, but the observable result, schema, parameter behavior, NULL semantics, and errors MUST match Appendix I. No client-visible rewrite is required.\
A server MAY accept text outside the Portable SQL Core as deployment behavior. A separately claimed Spark SQL 4.2 or other dialect profile defines that additional syntax, function resolution, command classification, and TCK corpus. The core protocol does not provide session-level dialect discovery or negotiation.

## 16.13 Excluded and internal nodes

A server receiving an excluded standard node may implement it as an extension or reject it with the Chapter 7 envelope. It MUST NOT erase the node, reinterpret it as a required relation, or expose cached/session-owned data across sessions.\
Presentation formatting is client or optional-profile behavior. Core conformance does not require exact `show` or HTML rendering.

## 16.14 Relation TCK requirements

The core TCK MUST:

* construct protobuf plans directly;\
* include at least one positive case for every required relation row and every required field-domain branch;\
* compare schema and rows, including empty input, nulls, NaN, duplicate names/rows, and required ordering;\
* test server rejection for invalid required fields, mutually exclusive alternatives, cross-session references, cycles, and unsupported nodes, except for the two client-only invalid shapes named in §16.1;\
* verify that required clients do not emit either §16.1 exception and assert no server outcome for those shapes;\
* cover exact-mode `NearestByJoin`, correlated `LateralJoin`, `AsOfJoin`, `WithRelations`, `Parse`, `Transpose`, NA, and every statistical relation; and\
* avoid physical-plan and partition-layout assertions unless an operator exposes them.

The core TCK tests SQL transport framing, parameter exclusivity, and successful/failed `SC-1.0-P1-PORTABLE-SQL` queries. Complete-dialect TCKs own all other SQL statement results and command classification.

## 16.15 Plan hierarchy

`ExecutePlanRequest` selects a `Plan`; the plan selects either a row-producing `Relation` tree or a side-effecting `Command`. Expressions appear inside relation and command fields. Every relation, expression, and command selects one proto-oneof variant.
