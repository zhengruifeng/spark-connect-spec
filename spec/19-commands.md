# 19. Commands

`Command` represents eager or side-effecting work submitted in an `ExecutePlanRequest.plan`. `command_type` selects exactly one command. Chapter 10 defines operation identity, response streaming, reattachment, cancellation, and retry hazards.

## 19.1 Common command contract

A command MUST select one supported variant and provide all required fields. Validation SHOULD occur before external side effects begin. Unsupported commands MUST fail explicitly; acknowledging and discarding a command is prohibited.\
For one accepted logical operation, the server executes the command once. Retrying under a new operation ID is a new execution and may repeat effects. Reattachment, when enabled, resumes response delivery and MUST NOT re-execute the command.\
Authentication and authorization for commands are deployment-specific and outside v1.0 conformance. UserContext, tags, runner names, paths, table names, and options have no standardized privilege meaning.

## 19.2 v1.0 profile

Required membership is fixed exclusively by manifest SC-1.0-P1 (§6.3). The required command variants are:

* `WriteOperation`;\
* `WriteOperationV2`;\
* `CreateDataFrameViewCommand`;\
* `SqlCommand`; and\
* `MergeIntoTableCommand.`

`CheckpointCommand`, `RemoveCachedRemoteRelationCommand`, and `GetResourcesCommand` are OPTIONAL. Resource profiles and external commands are outside the portable core. UDF registration, streaming, pipelines, and ML follow Chapters 18, 20, and 21.

## 19.3 `SqlCommand`

`SqlCommand` is the eager SQL-entry transport used by current Spark Connect clients. Its canonical v4.2.0 form carries an `input` relation, normally `Relation.SQL` or a `WithRelations` root containing it. The legacy fields `sql`, literal `args`, `pos_args`, `named_arguments`, and `pos_arguments` are deprecated but remain wire-compatible. Named and positional forms MUST NOT coexist where the proto forbids it.\
The Portable SQL Core contains queries only. For a portable query, the server MUST return `ExecutePlanResponse.SqlCommandResult.relation` as a relation representing that query; it MUST NOT invent an update count, classify the query as a side-effecting command, or eagerly collect it merely because it arrived in `SqlCommand`. Executing the returned relation produces the same schema and rows as executing the equivalent `Relation.SQL` plan.\
Full dialect profiles may add DDL, DML, scripts, and a query-versus-command classification. The separately named dialect profile MUST define when the returned relation is an opaque/local command result rather than a lazy SQL relation. That classification is not inferred into the Portable SQL Core.\
SQL parameters are typed expressions and MUST NOT be substituted into raw text. The Portable SQL Core binding and exclusivity rules are in Appendix I; parsing and execution also use the session catalog, namespace, configuration, and deployment context.

## 19.4 `CreateDataFrameViewCommand`

This command requires `input`, `name`, `is_global`, and `replace`.\
`is_global=false` creates a temporary view in the logical session. `is_global=true` creates a global temporary view in the logical `global_temp` namespace. `replace=false` returns the canonical already-exists condition when a view of the same scope exists. `replace=true` atomically replaces that view. A conflict with a table, permanent view, or object of the wrong scope returns the canonical wrong-object-type condition; it is never replaced.\
Session-local views follow Chapter 8 clone and release rules. Global temporary views are not copied into a clone as session-local state. This command is the typed Catalog-adjacent surface for DataFrame-backed view creation; `Catalog.cat_type` has no create-view operation.

## 19.5 `WriteOperation`

`WriteOperation` requires input and non-`UNSPECIFIED` `SaveMode`. It may specify source and exactly one save destination: path or table. For the five required providers, exactly one path or table destination is required. Destination-free providers such as jdbc or noop are outside the required provider profile.\
Table destination contains `table_name` and either `SAVE_AS_TABLE` or `INSERT_INTO`. The distinction is normative: insert-into uses the existing table contract and does not become save-as-table merely because creation would be convenient.\
Optional sort columns, partition columns, bucketing, clustering, options, and schema-evolution flag MUST be preserved. Bucketing requires positive bucket count and at least the required bucket columns. sort columns without bucketing, non-positive bucket counts, simultaneous partitioning and clustering, and every combination rejected by the canonical command row fail analysis.\
Save modes mean append, overwrite, error-if-exists, or ignore. UNSPECIFIED is invalid. Ignore suppresses only the ordinary existence condition; it does not suppress invalid data, deployment-policy, or provider failures.

## 19.6 `WriteOperationV2`

V2 writes require input, target `table_name`, and one mode:

* `CREATE`\
* `OVERWRITE`\
* `OVERWRITE_PARTITIONS`\
* `APPEND`\
* `REPLACE`\
* `CREATE_OR_REPLACE`

Optional provider, partitioning expressions, options, table properties, overwrite condition, clustering columns, and schema evolution are interpreted according to DataFrameWriterV2.\
An overwrite condition is valid only for the matching overwrite mode. Partition overwrite remains distinct from filter overwrite. Create, replace, and create-or-replace preserve their different existence and metadata behavior.

## 19.7 Merge

`MergeIntoTableCommand` requires target table name, source relation, merge condition, and schema-evolution flag. It carries ordered matched, not-matched, and not-matched-by-source action lists.\
Actions are `MergeAction` expressions defined in Chapter 17. Order is semantic. For each source-target pair, the first applicable clause in its category supplies the action. An unconditional clause must be last in its category. Invalid categories, misplaced or repeated unconditional clauses, unresolved assignments, and multiple source rows attempting to update or delete one target row MUST fail with the assigned canonical structured condition.\
Schema evolution applies only when requested and supported. It does not alter deployment access policy or silently coerce incompatible assignments.

## 19.8 Optional checkpoint and cache commands

`CheckpointCommand` requires relation, `local`, and `eager`; local checkpoints may carry `StorageLevel`. The result uses `CheckpointCommandResult`. Durable and local checkpoints have different fault-tolerance guarantees and MUST NOT be conflated.\
`RemoveCachedRemoteRelationCommand` requires the session-owned cached remote relation. Cross-session IDs MUST be rejected.\
These commands are OPTIONAL in v1. A server implementing them MUST follow cancellation, reattachment, and cleanup rules; a non-supporting server returns unsupported explicitly.

## 19.9 Resource and external commands

`GetResourcesCommand` and its resource map are OPTIONAL and Spark-specific. `CreateResourceProfileCommand` is outside the portable v1 core because it exposes Spark stage-level scheduling.\
`ExecuteExternalCommand names a runner, command string, and options. It is outside v1 and security-sensitive. Its authentication, authorization, runner policy, and command validation are entirely implementation-specific and are not TCK requirements.`

## 19.10 Deferred commands

The following are present in the proto but not required by v1:

* UDF, UDTF, and user-defined-data-source registration;\
* streaming start/query/manager/listener commands;\
* ML commands; and\
* pipeline commands.

Behavior of these deferred commands is outside v1 and cannot contribute to a conformance claim. Removing an unsupported extension and executing a child command is forbidden.

## 19.11 Results, errors, and partial effects

Command responses use the variant defined by the command and Chapter 10. A successful no-row command still terminates explicitly. Failure MUST NOT be represented as an empty successful result.\
The protocol does not make all commands transactional. Unless a canonical command row states an atomic post-condition, a failure may leave provider or catalog effects. A server MUST report the failure and MUST NOT claim portable rollback that the row does not guarantee.\
Options may contain credentials. Logs, warnings, and structured errors MUST redact secrets. Paths and identifiers in errors SHOULD be limited according to deployment policy.

## 19.12 Command TCK requirements

The core TCK MUST cover each save mode, path versus table destination, V2 mode validation, create/replace view behavior, portable-query SqlCommandResult.relation, typed named and positional SQL parameters, parameter exclusivity, merge clause ordering, retry/reattachment without duplicate effects, unsupported commands, and in-scope error propagation. It does not test command authorization.\
Query-versus-command classification, eager DDL/DML results, scripts, and update-like results belong to a separately claimed full-dialect TCK. They are not core `SqlCommand` pass conditions.
