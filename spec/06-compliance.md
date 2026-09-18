# 6. Compliance

This chapter identifies the features and behaviors that a Spark Connect server implementation (§6.2, §6.3) and a Spark Connect client implementation (§6.4) are required to support to claim conformance with Spark Connect Specification version 1.0.\
There is exactly one conformance level for the complete Required Profile of a claimed specification version. Optional and Deferred protocol surfaces may exist, but they lie outside that Required Profile and do not create partial-conformance tiers, per-feature badges, or capability subsets. Per-chapter, per-API, and per-module TCK results are diagnostic only. This specification defines no capability-discovery mechanism for selecting a partial profile.

## 6.1 Definitions

To avoid ambiguity, the following terms are used throughout this specification:

* **Spark Connect server implementation** — an executable that accepts incoming gRPC connections on a Spark Connect endpoint and serves the RPCs defined in §5.2.\
* **Spark Connect client implementation** — a library or program that opens a gRPC channel to a Spark Connect server and issues RPCs defined in §5.2.\
* **Reference implementation** — the Spark Connect server and clients included in the Apache Spark release that this specification version names as its reference Spark version. For v1.0 of this specification, the reference is Apache Spark 4.2.0.\
* **Required Profile — the complete set of interfaces and behaviors marked Required for a specification version. Conformance is claimed and evaluated only for this set as a whole.**\
* **Required interface — an interface (gRPC service, message type) that a conforming implementation MUST implement.**\
* **Required behavior** — a behavior that a conforming implementation MUST exhibit. Required behaviors are introduced with normative language ("MUST", "SHALL"). Where a required behavior is defined by reference to the reference implementation, it is the *observable* behavior of the reference implementation that is normative, not any particular code path.\
* **Fully implemented** — describes a Required interface for which the implementation provides correct behavior for every message variant, field, and RPC method, in accordance with the chapter in this specification that defines it.\
* **Supported feature** — describes a feature for which a conforming implementation provides standard syntax and semantics as defined in this specification.\
* **Wire extension — a feature that transmits a nonstandard protocol payload through an explicit google.protobuf.Any extension point defined by the Apache Spark proto. Wire extensions MUST NOT alter required semantics. An out-of-profile client API that lowers entirely to standard plans and RPCs is a client convenience API, not a wire extension, and does not require Any.**

### 6.1.1 Normative responsibility and precedence

The Spark Connect contract is split by concern. No source may silently expand the authority assigned to another source.

| Concern | Controlling authority |
| :---- | :---- |
| Required-profile membership, argument domains, negative cases, Arrow-mapping rows, cast-conversion rows, result-schema rows, and deprecation rows | The canonical JSON bundle `docs/spark-connect/spec/manifests/sc-1.0-p1.json` at its immutable Apache Spark publication commit, identified by the whole-file SHA-256 recorded in §22.1.1. |
| Protobuf message names, field numbers, field types, enum values, and RPC signatures | The proto IDL at the pinned Apache Spark reference commit. |
| Portable observable semantics | The normative prose of this specification. A bundle row may add a field domain or named constraint, but it need not repeat every rule in the controlling chapter. |
| Observable behavior expressly delegated by a canonical row or normative section | The Apache Spark reference implementation at the pinned reference commit. The delegation controls only the named question. |
| Conformance evidence | The TCK version that records the same profile ID, publication commit, bundle path, and whole-file digest. The TCK encodes the contract; it does not create or broaden it. |
| Navigation and generated mirrors | The Google Sheet, hosted API documentation, source line numbers, family indexes, and readable appendix tables are informative unless the canonical bundle expressly incorporates their content. |

The authorities are complementary, not a general “last document wins” chain. If two controlling authorities answer the same question inconsistently, the discrepancy blocks release. A finalized implementation MUST NOT choose the more convenient interpretation. Profile membership is never inferred from source discovery, runtime lowering, an implementation-specific test, or a live document outside the bundle digest. Normative chapter prose controls the behavior of every selected member even when the row cites the chapter without restating each rule; chapter prose does not make an unselected member required.\
For example:

* **Membership question:** “Is `NearestByJoin` required?” The answer comes from the relation member in the published bundle. Its presence in the IDL or Chapter 16 alone cannot make it required.\
* **Semantic question:** Once that member is selected, “Which row is returned when equal ranks cross the top-K boundary?” is answered by §16.5.3: any subset of the tied candidates that fills the remaining positions is valid. The prose controls even if the bundle row only cites §16.5.3.\
* **Wire-shape question:** “What are the field number and protobuf type of `num_results`?” is answered only by the pinned proto IDL. Neither prose nor a TCK may redefine them.

Silence is not a reference delegation. Phrases such as “follows Spark behavior,” “as Spark does,” or “reference semantics” do not create requirements. A valid delegation MUST identify the pinned reference version, the exact behavioral question, and the canonical row or normative section that delegates it. Otherwise this specification must state the portable behavior directly or classify it outside conformance. The finalization audit MUST reject an unqualified reference-behavior phrase in normative text.

## 6.2 Guidelines and Requirements

The following guidelines apply to all conforming Spark Connect server implementations:

* A Spark Connect server implementation MUST implement every RPC marked **Required for v1.0** in §5.2. RPCs marked **Deferred** in §5.2 MAY be implemented or return gRPC `UNIMPLEMENTED`.\
* A Spark Connect server implementation MUST execute every required Relation, Expression, Command, operator, function, and Portable SQL query with the semantics defined by its canonical row and controlling chapter. SQL behavior outside `SC-1.0-P1-PORTABLE-SQL` comes only from a separately claimed dialect profile or out-of-profile deployment behavior. The pinned reference implementation supplies behavior only where this specification or a canonical row expressly delegates to it.\
* A Spark Connect server implementation MUST return errors via the structures defined in Chapter 7. Returning a different error structure, an unstructured string, or no error at all where one is expected is a conformance violation.\
* A Spark Connect server implementation MUST honor logical-session isolation: changes to session-scoped state (configuration, temporary views, registered artifacts) in one session\_id MUST NOT be observable from a different session\_id unless explicitly shared by an out-of-band mechanism. This requirement does not define or certify isolation between authenticated principals or tenants.\
* A Spark Connect server implementation MUST accept and ignore unknown fields in client request messages, in accordance with proto3 unknown-field handling, *except* where this specification explicitly defines an unknown-field response.\
* A Spark Connect server implementation MUST NOT advertise a feature it does not implement. There is no protocol mechanism for advertising features (Goal 2); a server's claim of Spark Connect 1.0 conformance is itself the advertisement, and it MUST be true.\
* A Spark Connect server implementation MAY provide wire extensions through the explicit google.protobuf.Any fields listed in §22.3. Wire extensions MUST NOT alter conformance behavior; an implementation that depends on one to pass the TCK is non-conforming.

## 6.3 Spark Connect 1.0 API Compliance

Required Profile `SC-1.0-P1` is the sole normative source of interface membership for Spark Connect 1.0. Final publication will commit one deterministic JSON file at `docs/spark-connect/spec/manifests/sc-1.0-p1.json` in `apache/spark` and record its immutable publication commit and whole-file SHA-256.\
**Reference baseline:** Apache Spark tag v4.2.0, commit 32f7299601108917fb01920a54e084595b7b3bf8. Source discovery at that commit supplies candidates and provenance; only canonical rows create requirements.\
Authentication and authorization are not members of SC-1.0-P1. The profile does not define credential formats, principal derivation, identity propagation, access-control policy or decisions, or cross-principal isolation. The TCK runs in a pre-authorized environment. Passing it is evidence of protocol conformance, not security certification. UserContext has no standardized security meaning.

| Bundle member | Contents |
| :---- | :---- |
| `SC-1.0-P1-WIRE` | Required RPCs, operations, messages, plan nodes, data types, configuration keys, session-lifecycle rows, and error-registry pin. |
| `SC-1.0-P1-CLIENT` | Required Python and Scala symbols and overloads, argument constraints, and unsupported cases. |
| `SC-1.0-P1-PROVIDER` | Required data-source providers, capabilities, options, defaults, and failure behavior. |
| `SC-1.0-P1-FUNCTIONS` | The closed `UnresolvedFunction` operator and function rows defined by Appendix C. |
| `SC-1.0-P1-EXPRESSION-SYNTAX` | The restricted `ExpressionString` productions defined by Appendix I. |
| `SC-1.0-P1-PORTABLE-SQL` | The query-statement, lexical, parameter, type-alias, function-reference, semantic, rejection, and TCK rows defined by Appendix I. |
| `SC-1.0-P1-ARROW` | Arrow mapping rows, mode selectors, nested-type rules, and TCK cases. |
| `SC-1.0-P1-CASTS` | Cast source, target, mode, conversion, boundary, and Portable SQL alias-binding rows. |
| `SC-1.0-P1-CATALOG-SCHEMAS` | Ordered result-schema rows for tabular Catalog operations. |
| `SC-1.0-P1-DEPRECATIONS` | Deprecation rows, including the defined meaning of an empty array. |

The Google Sheet is an informative source-inventory view for reviewing SC-1.0-P1. Its currently displayed snapshot has not been regenerated from a published canonical bundle and is not v1.0 membership or publication evidence. It cannot add a symbol, overload, function, syntax form, or wire node. Source paths, generated API pages, and appendix indexes are also informative.

| Surface | Required core | Outside the Required Profile |
| :---- | :---- | :---- |
| gRPC RPCs | `ExecutePlan`, `AnalyzePlan`, `Config`, `Interrupt`, `ReleaseSession`, `FetchErrorDetails`, `CloneSession`, `GetStatus` | Optional: `ReattachExecute`, `ReleaseExecute`. Deferred: artifact RPCs. |
| Analyze operations | `Schema`, `Explain`, `TreeString`, `IsLocal`, `IsStreaming`, `InputFiles`, `SparkVersion`, `DDLParse` | Optional operations listed in Chapter 9. |
| Data types | `Null`, `Byte`, `Short`, `Integer`, `Long`, `Float`, `Double`, `Decimal`, `Boolean`, `String`, `Binary`, `Date`, `Timestamp`, `TimestampNTZ`, `Array`, `Map`, `Struct` | Character, interval, Variant, UDT, Unparsed, Time, geospatial, and nanosecond timestamp types. |
| Relations | Every standard portable batch variant enumerated in §16.2: `Read`, `Project`, `Filter`, `Join`, `SetOperation`, `Sort`, `Limit`, `Aggregate`, `SQL` transport, `LocalRelation`, `Sample`, `Offset`, `Deduplicate`, `Range`, `SubqueryAlias`, `Repartition`, `ToDF`, `WithColumnsRenamed`, `Drop`, `Tail`, `WithColumns`, `Hint`, `Unpivot`, `ToSchema`, `RepartitionByExpression`, `CollectMetrics`, `Parse`, `AsOfJoin`, `WithRelations`, `Transpose`, `LateralJoin`, `NearestByJoin`, the NA/statistical variants, and required `Catalog` operations. | ML; worker/UDF; streaming; extension/unknown; vendor-specific `RelationChanges`; dialect-owned `UnresolvedTableValuedFunction`; `ShowString`/`HtmlString`; and cached-relation transport nodes. |
| Expressions | `Literal`, `UnresolvedAttribute`, constrained `UnresolvedFunction`, constrained `ExpressionString`, `UnresolvedStar`, `Alias`, `Cast`, `UnresolvedRegex`, `SortOrder`, `LambdaFunction`, `Window`, `UnresolvedExtractValue`, `UpdateFields`, `UnresolvedNamedLambdaVariable`, `NamedArgumentExpression`, `SubqueryExpression`; `MergeAction` only in required merge commands. | `CommonInlineUserDefinedFunction`, `CallFunction`, `TypedAggregateExpression`, `DirectShufflePartitionID`, and extensions. |
| Unresolved functions | Operators `+`, `-`, `*`, `/`, `%`, `==`, `!=`, `<`, `<=`, `>`, `>=`, `<=>`, `and`, `or`, `not`, `isNull`, `isNotNull`, `negative`; functions `abs`, `coalesce`, `nullif`, `lower`, `upper`, `length`, `substring`, `substr`, `concat`, `trim`, `count`, `sum`, `avg`, `min`, `max`, `transform`. | Every other registered function, all UDFs, and every invocation without an applicable canonical overload row. |
| Expression text | The closed grammar in Appendix I: literals, attributes, parentheses, arithmetic, comparisons, Boolean logic, null tests, `CASE`, primitive casts, calls to the required function set, and lambdas used by `transform`. | Statements, subqueries, windows, DDL/DML, arbitrary function names, and every unlisted expression production. |
| SQL text | Relation.SQL and SqlCommand framing plus SC-1.0-P1-PORTABLE-SQL: SELECT/VALUES, the closed clause/expression/function/type-alias subset, typed parameters, and query-result behavior in Appendix I. | DDL, DML, scripts, complete Spark SQL, and every unlisted statement, production, operator, type name, or function. Complete dialects use separate optional profiles. |
| Commands | `WriteOperation`, `WriteOperationV2`, `CreateDataFrameViewCommand`, `SqlCommand`, `MergeIntoTableCommand` | UDF registration, streaming, ML, pipelines, resource profiles, cache/checkpoint helpers, resources, and external commands. |
| Providers | `parquet`, `orc`, `json`, `csv`, and `text` with Appendix H constraints; V2 table writes use `parquet`. | Providers and options absent from Appendix H. |
| Errors and results | Chapters 7, 10, 14, and 15, including the pinned error registry and Arrow mappings. | Later registry additions and implementation-only diagnostics. |

Each chapter defines semantics for rows assigned to it; no chapter independently expands membership. Every positive and negative TCK case MUST cite canonical row IDs. Core relation and expression tests MUST construct protobuf plans directly. Core SQL-text tests are limited to `SC-1.0-P1-PORTABLE-SQL`; complete Spark SQL or other dialect tests run only for the separately claimed dialect profile. `ExpressionString` tests remain part of the core TCK.\
A later Spark release may claim 1.0 conformance by passing this frozen profile. Adding a required interface, relation, expression, function, syntax form, provider, client overload, or error row requires a new specification minor version and canonical bundle.

## 6.4 Client Compliance and the Core SQL API

A conforming Spark Connect 1.0 client MUST implement every applicable `CLI-*` row in the canonical bundle's `client.overloads` member. A row is required only for its named symbol, exact signature, and referenced argument constraints. Appendix G and the Google Sheet are generated views and cannot change membership.\
A conforming client MUST:

* provide the required session, RuntimeConfig, Catalog, DataFrame, expression, read, write, action, NA, and statistics rows;\
* emit only required wire nodes for required rows;\
* restrict `UnresolvedFunction` and `ExpressionString` to the Appendix C and I manifests when executing core conformance cases;\
* transmit Portable SQL Core text and typed parameters without substitution or dialect rewriting when executing core conformance cases;\
* treat `SparkSession.sql` as an SQL-text transport API and not as a promise that every server implements the complete Spark SQL dialect;\
* preserve the session, error, execution, parameter-binding, and Arrow-result contracts; and\
* fail explicitly for locally detectable unsupported cases. The client MUST NOT substitute another provider, remove an argument, rewrite SQL into a different statement, or discard an unsupported plan wrapper.

The core client manifest MUST NOT require an API whose only standard lowering uses an excluded relation, expression, function, or command. In particular, presentation APIs that require `ShowString` and function wrappers outside Appendix C are optional client conveniences unless a separate profile promotes them.\
A client need not parse statement SQL. It sends named or positional parameters as typed expressions and MUST NOT bind them by string substitution. A conforming server accepts the Portable SQL Core regardless of its native parser or backend engine. Acceptance of text outside that core is not a core conformance result; a claimed dialect profile supplies those tests.\
Spark Connect Specification 1.0 defines no dialect-discovery or dialect-negotiation RPC. A client therefore has no conformance obligation to discover, expose, or bind optional full-dialect metadata to a session. It MAY display deployment metadata supplied out of band or accept an application-selected dialect mode, but absence of that metadata MUST NOT prevent Portable SQL Core execution.\
Python and Scala are the official API-parity languages. Their required module/type names, method names, signatures, defaults, and return shapes are exactly the canonical client rows. Public API documentation and source paths are provenance only.\
A client MAY expose additional APIs. An API that lowers entirely to standard wire nodes is an out-of-profile convenience API. An API that sends a nonstandard payload is a wire extension and MUST use an explicit `google.protobuf.Any` field. Neither can satisfy a required client row.\
Clients in other languages may expose an idiomatic DataFrame API. Their conformance is assessed at the wire level. Servers have no Python or Scala signature obligation.

## 6.5 Determining Compliance Level

Conformance to Spark Connect 1.0 is established by passing the Spark Connect TCK at an immutable commit that identifies the same profile and bundle digest as the conformance report. The TCK assumes a pre-authorized endpoint and does not test or certify authentication, authorization, or cross-principal security.\
There is no self-certification mechanism, third-party certification authority, or compliance branding administered by the specification authors. The implementer makes the conformance claim, the TCK result supplies the evidence, and downstream consumers verify the claim against the published output.

## 6.6 Conformance report and implementation descriptor

Every published conformance run MUST emit a machine-readable report. The report is evidence for one complete profile, not a capability-negotiation response and not a menu of independently claimable modules.

| Field | Required value |
| :---- | :---- |
| `specification_version` | The claimed Spark Connect specification version. |
| `profile_id` | `SC-1.0-P1` for this version. |
| `portable_sql_profile_id` | `SC-1.0-P1-PORTABLE-SQL`. |
| `bundle_publication_commit`, `bundle_path`, `bundle_sha256` | The exact Apache Spark Git pin, canonical path, and whole-file digest recorded by §22.1.1. |
| `reference_spark_tag`, `reference_spark_commit` | The reference implementation snapshot. |
| `implementation_name`, `implementation_version` | The tested product identity. |
| `tck_commit` | The immutable TCK revision used for the run. |
| `overall_result` | Exactly PASS only when every applicable Required Profile case passes, including every descriptor-declared LIFE-SESSION-EVICT-\* row. |
| `test_results` | Row-addressed diagnostic results that cite canonical manifest row IDs. |

An implementation MAY additionally disclose extensions and operational limits such as inbound message size, preferred Arrow chunk size, idle-session timeout, and reattachment retention. When it accepts text beyond the Portable SQL Core, it MAY report `sql_dialect_id`, `sql_dialect_version`, and an immutable dialect-profile ID and digest. These are optional deployment-evidence fields, not endpoint/session metadata and not a discovery protocol. They MUST NOT reclassify a required row, turn a failed core test into a pass, or create a partial Spark Connect 1.0 claim.

### 6.6.1 TCK deployment descriptor and adapter contract

#### 6.6.1.1 Purpose and ownership

The deployment descriptor and adapter control the TCK fixture. They are not Spark Connect protocol messages, endpoint capability discovery, production configuration, or portable query semantics. They record how one test run restarts the server and, when declared, induces a supported session-eviction event. They cannot add or remove a Required Profile row or change the meaning of a row.\
This section defines the evidence that binds the fixture to a conformance report. The TCK repository at the report's immutable `tck_commit` MUST publish machine-readable schemas for `SC-TCK-DEPLOYMENT-1` and `SC-TCK-ADAPTER-1` that implement this section. Those schemas may reject malformed fixture data; they cannot create Spark Connect requirements. Chapters 7–21 remain the authority for observable client-server behavior.

#### 6.6.1.2 Deployment descriptor

Each run MUST publish one JSON descriptor with exactly the following ten properties, each once, and no other property. A duplicate property name invalidates the descriptor.

| Property | Required value |
| :---- | :---- |
| `adapter_artifact_digest` | `sha256:` followed by 64 lowercase hexadecimal digits identifying the immutable adapter artifact invoked for the run. |
| `adapter_id` | A non-empty ASCII string matching `^[A-Za-z][A-Za-z0-9._-]{0,63}$`. |
| `adapter_protocol_version` | Exactly `SC-TCK-ADAPTER-1`. |
| `adapter_version` | A non-empty ASCII string matching `^[0-9A-Za-z][0-9A-Za-z._+-]{0,63}$`. |
| `optional_actions` | `[]` when `session_eviction_reasons` is empty; otherwise exactly `["evict_session"]`. |
| `profile_id` | The report's exact `profile_id`; `SC-1.0-P1` for this specification. |
| `required_actions` | Exactly `["restart_server", "wait_ready"]`. |
| `schema_version` | Exactly `SC-TCK-DEPLOYMENT-1`. |
| `session_eviction_reasons` | A duplicate-free, lexicographically sorted subset of `IDLE_TIMEOUT` and `RESOURCE_PRESSURE`. |
| `tck_commit` | The report's exact TCK commit as a 40-character lowercase Git object ID matching `^[0-9a-f]{40}$`. |

#### 6.6.1.3 Canonical bytes and digest

The descriptor file is the UTF-8 output of the JSON Canonicalization Scheme in RFC 8785 applied to the descriptor object. It has no byte-order mark, leading or trailing bytes, or final line terminator. RFC 8785 controls property order, string escaping, number rendering, and whitespace; this specification does not define a second JSON serialization algorithm. A producer MUST reject duplicate property names before canonicalization. The descriptor SHA-256 covers exactly the RFC 8785 output bytes.\
The descriptor contains no timestamp, credential, absolute machine path, or environment-specific secret. Its variable string fields use the ASCII grammars above, so Unicode normalization cannot change their identity.

#### 6.6.1.4 Adapter process protocol

The runner receives the adapter executable through its local `--deployment-adapter` option. For each action it starts one process, writes one UTF-8 JSON request followed by LF to standard input, closes standard input, and reads one UTF-8 JSON response followed by LF from standard output. The adapter writes diagnostics only to standard error. The local executable path is not portable evidence; `adapter_artifact_digest` binds the invoked artifact.\
An `SC-TCK-ADAPTER-1` request contains exactly four properties: string `protocol_version` equal to `SC-TCK-ADAPTER-1`; string `request_id` unique within the run and matching the canonical lowercase UUID form `^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$`; string `action` declared by the descriptor; and object `arguments` defined by the selected action. An `OK` response contains exactly string `protocol_version`, the exact string `request_id`, and string `status` equal to `OK`. An `ERROR` response adds non-empty string `error_code` and `message` and sets `status` to `ERROR`. Missing, duplicate, additional, or incorrectly typed properties invalidate a request, response, or nested `arguments` object.

| Action | Exact `arguments` object | `OK` completion point |
| :---- | :---- | :---- |
| `restart_server` | `{}` | The prior process can no longer serve requests and replacement startup has begun. |
| `wait_ready` | Exactly one positive JSON integer property, `timeout_ms`. | The replacement at the tested endpoint accepts Spark Connect RPCs before the timeout. |
| `evict_session` | Exactly string `session_id` in canonical lowercase UUID form and string `reason` declared in `session_eviction_reasons`. | The identified live session is evicted and the §8.4.3.2 post-condition is observable. |

#### 6.6.1.5 Validation and report binding

A malformed request or response, mismatched `request_id`, undeclared action or eviction reason, nonzero process exit, timeout, or `ERROR` response makes the affected TCK case `ERROR`. The runner MUST NOT convert it to `NOT_APPLICABLE` or silently skip the case.\
The conformance report MUST record `tck_deployment_descriptor_schema_version`, `tck_deployment_descriptor_path`, `tck_deployment_descriptor_sha256`, `adapter_protocol_version`, and `adapter_artifact_digest`. The descriptor path identifies the artifact published with the report. `tck_deployment_descriptor_sha256` is 64 lowercase hexadecimal characters without a prefix. `adapter_artifact_digest` is identical to the descriptor value. The descriptor's `profile_id`, `tck_commit`, adapter protocol version, and adapter artifact digest MUST match the report byte-for-byte.

#### 6.6.1.6 Lifecycle-row applicability

The Required Profile contains `LIFE-SESSION-EVICT-IDLE-1`, applicable exactly when `session_eviction_reasons` contains `IDLE_TIMEOUT`, and `LIFE-SESSION-EVICT-RESOURCE-1`, applicable exactly when it contains `RESOURCE_PRESSURE`. The descriptor selects applicability of these existing rows; it does not change membership or semantics.\
Each applicable row MUST be reported as `PASS`, `FAIL`, or `ERROR`; `FAIL` or `ERROR` prevents `overall_result=PASS`. Each undeclared row MUST be reported as `NOT_APPLICABLE` with `applicability_reason="UNDECLARED_SESSION_EVICTION_REASON"` and the absent enum value. An undeclared row does not gate the result.
