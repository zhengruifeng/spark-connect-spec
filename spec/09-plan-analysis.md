# 9. Plan Analysis

This chapter describes the `AnalyzePlan` RPC, which allows a client to inspect a plan — or manipulate its caching state — without executing it. `AnalyzePlan` is a unary RPC; one request produces one response:\
`rpc AnalyzePlan(AnalyzePlanRequest) returns (AnalyzePlanResponse) {}`\
Plan analysis is where Spark Connect's lazy-analysis model becomes observable. Clients build plans without server contact, so a plan is first validated when it reaches the server — either through `ExecutePlan` (Chapter 10\) or through this RPC. Client API surfaces such as `df.schema`, `df.printSchema()`, `df.explain()`, `df.isLocal`, and `df.isStreaming` are implemented as `AnalyzePlan` operations, and analysis errors (unresolved columns, type mismatches) surface on these calls rather than at DataFrame construction time.\
Required versus optional operation membership is fixed exclusively by manifest SC-1.0-P1 (§6.3). The request carries a oneof analyze field selecting exactly one of fourteen operations. The manifest requires Schema, Explain, TreeString, IsLocal, IsStreaming, InputFiles, SparkVersion, and DDLParse; it classifies SameSemantics, SemanticHash, Persist, Unpersist, GetStorageLevel, and JsonToDDL as OPTIONAL. A server that does not implement an OPTIONAL operation MUST fail the request with a clear error (gRPC UNIMPLEMENTED or the structured terminal gRPC status defined by Chapter 7); it MUST NOT return an empty or default-valued result variant.

## 9.1 The `AnalyzePlan` Request Envelope

`message AnalyzePlanRequest {`\
  `// (Required) Session identifier, set by the client (UUID format).`\
  `string session_id = 1;`

  `// (Optional) Server-side session id last observed by the client.`\
  `optional string client_observed_server_side_session_id = 17;`

  `// (Required) User context`\
  `UserContext user_context = 2;`

  `// (Optional) Client language/version string, for logging purposes`\
  `// only; will not be interpreted by the server.`\
  `optional string client_type = 3;`

  `oneof analyze {`\
    `Schema schema = 4;`\
    `Explain explain = 5;`\
    `TreeString tree_string = 6;`\
    `IsLocal is_local = 7;`\
    `IsStreaming is_streaming = 8;`\
    `InputFiles input_files = 9;`\
    `SparkVersion spark_version = 10;`\
    `DDLParse ddl_parse = 11;`\
    `SameSemantics same_semantics = 12;`\
    `SemanticHash semantic_hash = 13;`\
    `Persist persist = 14;`\
    `Unpersist unpersist = 15;`\
    `GetStorageLevel get_storage_level = 16;`\
    `JsonToDDL json_to_ddl = 18;`\
  `}`

  `// Nested operation messages are specified in §9.2 through §9.8.`\
`}`\
Envelope rules:

* `session_id identifies the logical session within the server's deployment-defined scope; user_context has no standardized security meaning. The first AnalyzePlan on an unknown key materializes the session (§8.4.1).`\
* `client_observed_server_side_session_id` (field 17\) — when present, the server MUST validate it against the current session realization and fail with `INVALID_HANDLE.SESSION_CHANGED` on mismatch (§8.1).\
* `client_type` (field 3\) — logging only; the server MUST NOT let it influence behavior.\
* **Operation selection** — exactly one `analyze` variant MUST be set. The server MUST answer with the matching `result` variant (§9.9).\
* **Side effects** — every operation except `Persist` and `Unpersist` MUST be free of side effects on session-scoped state. `Persist` and `Unpersist` modify caching state (§9.8) but MUST NOT execute the plan.\
* **Plan-typed inputs** — where an operation carries a `Plan`, the plan MUST carry a relation (`Plan.root`); a server SHOULD reject a plan carrying a `Command` for an analysis operation. Analysis failures (unresolved attributes, type errors, parse errors) are application errors delivered per Chapter 7.

## 9.2 Retrieving Schema (required for v1.0)

`Schema` is the worked example for this chapter; the remaining operations follow the same request/response shape.\
`// AnalyzePlanRequest.Schema`\
`message Schema {`\
  `// (Required) The logical plan to be analyzed.`\
  `Plan plan = 1;`\
`}`\
`// AnalyzePlanResponse.Schema`\
`message Schema {`\
  `DataType schema = 1;`\
`}`\
The server MUST run analysis sufficient to fully resolve the plan, including name resolution, type checking, and type coercion, and return the plan's output schema. It MUST NOT execute the plan to produce result rows or command side effects. It MAY read source data, source metadata, or an input relation only to the extent required to determine the schema. Examples are implicit pivot-value discovery (§16.6.4), schema inference for a schema-less required data-source read (Appendix H), and `Parse` schema inference (§16.10). These examples do not form an exhaustive registry; the purpose restriction in this paragraph governs every analysis-time read.\
Response contract:

* `schema` MUST be present on success and MUST be a `DataType` carrying a `struct` kind (Chapter 15\) whose fields describe the plan's output columns in order.\
* Field names MUST be the resolved output names after aliasing. Field types and nullability MUST match execution of the same plan. A schema reported by `AnalyzePlan` that disagrees with the Arrow schema later delivered by `ExecutePlan` for the same plan is a conformance violation (see Chapter 14).\
* An analysis-time read MUST NOT expose result rows to the client, execute a command, mutate session-scoped state, or perform work unrelated to schema determination. It MUST honor the applicable operator or provider inference controls, including discovery limits, sampling ratios, and format options.\
* On analysis failure, the server MUST return an application error (Chapter 7), such as an `UNRESOLVED_COLUMN.*` error class for an unresolvable reference, and MUST NOT return a partial schema.\
* The core TCK MUST cover implicit pivot discovery, schema-less required-provider inference using metadata and sampled data, and schema-less `Parse` inference. Each case asserts the resolved schema, applicable inference controls, absence of result rows, and absence of command or session-state side effects.

## 9.3 Plan Properties: `IsLocal`, `IsStreaming`, `InputFiles` (required for v1.0)

Each operation takes one required `Plan plan` and returns its named response variant. The proto IDL controls the response field types: Boolean `is_local`, Boolean `is_streaming`, and repeated String `files`.

* `IsLocal` returns true exactly when the analyzed root is a `LocalRelation` or `CommandResult`. It returns false for `Range`, file or table reads, and every other root, even if a deployment happens to execute the plan in one process. The probe MUST NOT execute the plan.\
* `IsStreaming` returns true exactly when the plan contains a streaming source. A server that does not implement deferred Streaming still implements this probe and returns false for every accepted plan.\
* `InputFiles` returns a best-effort snapshot of file-source paths. Non-file plans and plans for which the server has no file snapshot return an empty list, not an error. The response variant remains present.

`GetStorageLevel`, the fourth property probe, is optional with `Persist` and `Unpersist` under §9.8.

## 9.4 Plan Explanation: `Explain` and `TreeString` (required for v1.0)

`Explain` takes a required plan and one non-unspecified `ExplainMode`; its response contains non-empty `explain_string`.\
The server rejects `EXPLAIN_MODE_UNSPECIFIED`. The named modes request:

* `SIMPLE`: the physical plan;\
* `EXTENDED`: parsed, analyzed, optimized, and physical stages;\
* `CODEGEN`: generated code, when the engine has it, plus the physical plan;\
* `COST`: the optimized logical plan with available statistics; and\
* `FORMATTED`: a physical-plan outline followed by node details.

Explain text describes the server's own planner. A non-Spark engine renders equivalent stages from its planner and may state that code generation or cost information does not apply. Clients MUST NOT parse explain text for portable control flow, and the TCK asserts mode coverage and non-empty output rather than Spark-specific wording.\
`TreeString` renders the analyzed plan's output schema, not the plan tree. The response contains non-empty `tree_string`. Its first line is `root`. Each field then appears in schema order as `|-- name: type (nullable = true|false)`; every nested level adds `|` followed by three spaces before `--`. Type text uses the required DDL spelling. With `level=N`, `N >= 1`, output stops after N nesting levels; an omitted level renders the full schema.

## 9.5 `SparkVersion` (required for v1.0)

The request variant is empty:\
// AnalyzePlanRequest.SparkVersion\
message SparkVersion {}\
The Apache Spark response at the reference version contains one field:\
// AnalyzePlanResponse.SparkVersion\
message SparkVersion {\
  string version \= 1;\
}\
`version` MUST be populated with the Spark version reported by the reference server. For a conforming alternative engine, it identifies the Apache Spark compatibility version claimed by the server, not an unrelated product, Java, Scala, or distribution version.\
The Apache Spark proto does **not** define `java_version`, `scala_version`, `dbr_version`, or a vendor field range in this message. Product/runtime metadata may be exposed only through an explicit extension point elsewhere in the protocol or an out-of-band mechanism. Implementations MUST NOT add unregistered fields to `AnalyzePlanResponse.SparkVersion` and describe them as part of Spark Connect v1.0.

## 9.6 Schema-String Conversion: `DDLParse` (required) and `JsonToDDL` (OPTIONAL)

These two operations convert between schema representations; neither takes a plan.\
`// AnalyzePlanRequest.DDLParse`\
`message DDLParse {`\
  `// (Required) The DDL formatted string to be parsed.`\
  `string ddl_string = 1;`\
`}`

`// AnalyzePlanResponse.DDLParse`\
`message DDLParse {`\
  `DataType parsed = 1;`\
`}`

* `DDLParse` parses a schema string in Spark SQL DDL syntax — a column list (`a INT, b STRING`) or a single type (`struct<a:int,b:string>`, `map<string,int>`, `decimal(10,2)`) — into a `DataType`. `parsed` MUST be present on success. On a malformed input the server MUST return an application error (the reference uses the `PARSE_SYNTAX_ERROR` error class) rather than a partial result. Clients use this operation to interpret user-supplied schema strings (e.g., the `schema(...)` argument of reader APIs) consistently with the server's own parser.

`// AnalyzePlanRequest.JsonToDDL`\
`message JsonToDDL {`\
  `// (Required) The JSON formatted string to be converted to DDL.`\
  `string json_string = 1;`\
`}`

`// AnalyzePlanResponse.JsonToDDL`\
`message JsonToDDL {`\
  `string ddl_string = 1;`\
`}`

* `JsonToDDL` (OPTIONAL in v1.0) converts a schema in the JSON format produced by `StructType.json` into the equivalent DDL string. `ddl_string` MUST be present on success; malformed JSON MUST produce an application error.

## 9.7 Semantic Comparison: `SameSemantics` and `SemanticHash` (OPTIONAL in v1.0)

`// AnalyzePlanRequest.SameSemantics`\
`message SameSemantics {`\
  `// (Required) The plan to be compared.`\
  `Plan target_plan = 1;`

  `// (Required) The other plan to be compared.`\
  `Plan other_plan = 2;`\
`}`

`// AnalyzePlanResponse.SameSemantics`\
`message SameSemantics {`\
  `bool result = 1;`\
`}`

`// AnalyzePlanRequest.SemanticHash`\
`message SemanticHash {`\
  `// (Required) The logical plan to get a hashCode.`\
  `Plan plan = 1;`\
`}`

`// AnalyzePlanResponse.SemanticHash`\
`message SemanticHash {`\
  `int32 result = 1;`\
`}`

* `SameSemantics` returns true if and only if the two plans are semantically equal under the server's canonical plan form — equal plans return the same results. The reference implementation compares Catalyst canonicalized plans; a non-Spark engine compares its own canonical form. False negatives are permitted (canonicalization is conservative); false positives are not: a server MUST NOT report two plans as equivalent when they can return different results.\
* `SemanticHash` returns an integer hash of the canonical plan form. Within one server, plans that compare equal under `SameSemantics` MUST have equal `SemanticHash` values.\
* Both judgments are engine-internal. Equivalence verdicts and hash values MUST NOT be compared across implementations, and MAY change across server versions. Clients use these for within-session plan deduplication only.

## 9.8 Caching Operations: `Persist`, `Unpersist`, `GetStorageLevel` (OPTIONAL in v1.0)

These are the only `AnalyzePlan` operations with side effects, and the only ones whose input is a bare `Relation` rather than a `Plan`:\
`// AnalyzePlanRequest.Persist`\
`message Persist {`\
  `// (Required) The logical plan to persist.`\
  `Relation relation = 1;`

  `// (Optional) The storage level.`\
  `optional StorageLevel storage_level = 2;`\
`}`

`// AnalyzePlanRequest.Unpersist`\
`message Unpersist {`\
  `// (Required) The logical plan to unpersist.`\
  `Relation relation = 1;`

  `// (Optional) Whether to block until all blocks are deleted.`\
  `optional bool blocking = 2;`\
`}`

`// AnalyzePlanRequest.GetStorageLevel`\
`message GetStorageLevel {`\
  `// (Required) The logical plan to get the storage level.`\
  `Relation relation = 1;`\
`}`

`// AnalyzePlanResponse variants: Persist and Unpersist are empty`\
`// messages; GetStorageLevel carries the result.`\
`message GetStorageLevel {`\
  `// (Required) The StorageLevel as a result of get_storage_level request.`\
  `StorageLevel storage_level = 1;`\
`}`

* Persist marks the relation for caching at the given StorageLevel. If storage\_level is absent, it uses the session's spark.sql.defaultCacheStorageLevel, whose default is MEMORY\_AND\_DISK. This configuration key and the caching operations remain optional in v1.0. Marking is lazy: the operation MUST NOT execute the plan; materialization happens on the next execution that covers the relation.\
* `Unpersist` removes the caching mark and releases materialized cache data. When `blocking = true`, the server MUST NOT respond until the cached data has been released.\
* GetStorageLevel returns the relation's current storage level; for a relation that is not cached, the server MUST return the NONE level, with all storage flags false, rather than an error.\
* For the `Persist`/`Unpersist` response variants, the presence of the (empty) matching variant in `result` is the success signal.\
* The StorageLevel memory, disk, off-heap, deserialization, and replication fields are physical caching hints in this optional surface. An engine MAY treat them as advisory, but MUST preserve the observable contract: a persisted-then-queried relation returns the same results, and GetStorageLevel round-trips the level it accepted.

## 9.9 The `AnalyzePlanResponse` Contract

`// Next ID: 16`\
`message AnalyzePlanResponse {`\
  `string session_id = 1;`

  `// Server-side generated idempotency key that the client can use to`\
  `// assert that the server side session has not changed.`\
  `string server_side_session_id = 15;`

  `oneof result {`\
    `Schema schema = 2;`\
    `Explain explain = 3;`\
    `TreeString tree_string = 4;`\
    `IsLocal is_local = 5;`\
    `IsStreaming is_streaming = 6;`\
    `InputFiles input_files = 7;`\
    `SparkVersion spark_version = 8;`\
    `DDLParse ddl_parse = 9;`\
    `SameSemantics same_semantics = 10;`\
    `SemanticHash semantic_hash = 11;`\
    `Persist persist = 12;`\
    `Unpersist unpersist = 13;`\
    `GetStorageLevel get_storage_level = 14;`\
    `JsonToDDL json_to_ddl = 16;`\
  `}`\
`}`\
Every response carries:

* `session_id` — MUST echo the `session_id` from the request.\
* `server_side_session_id` — MUST be populated. Clients MAY check it against prior responses to detect a server-side session restart (§8.1).\
* `result` — exactly one variant, and it MUST be the variant matching the request's `analyze` selection (`Schema` request → `Schema` result, and so on). Populating a different variant, or none, is a conformance violation. On failure no `AnalyzePlanResponse` is returned at all; the error travels per Chapter 7.

Per-operation success requirements are specified in §9.2-§9.8.\
---
