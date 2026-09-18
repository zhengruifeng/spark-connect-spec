# 22. Versioning and Evolution

Section 6 is the sole authority for conformance scope, status, and the Required Profile. This chapter defines only version identifiers, publication pins, finalization, and evolution; it cannot add or remove a required interface. Appendix K provides the informative contributor procedure for coordinating changes; it adds no versioning or finalization requirement.

## 22.1 Specification Versioning

The Spark Connect Specification uses MAJOR.MINOR numbers independently from Apache Spark release numbers, while every finalized specification version binds to an immutable canonical profile bundle and an immutable Apache Spark reference snapshot.

* A **major** version MAY introduce incompatible behavior through an explicitly approved process.\
* A **minor** version is additive within a major line and publishes a newly identified canonical bundle.\
* Editorial draft numbers such as v0.6 identify document revisions; they do not change the claimed specification version.

### 22.1.1 Frozen v1.0 reference snapshot

Spark Connect Specification 1.0 is pinned to:

| Item | Frozen value |
| :---- | :---- |
| Required Profile | SC-1.0-P1 in §6.3 |
| Apache Spark reference release | `v4.2.0` |
| Apache Spark reference commit | `32f7299601108917fb01920a54e084595b7b3bf8` |
| Canonical bundle path | docs/spark-connect/spec/manifests/sc-1.0-p1.json |
| Bundle publication commit | TBD — finalization blocker |
| Whole-bundle SHA-256 | TBD — finalization blocker |
| Proto IDL provenance | sql/connect/common/src/main/protobuf/spark/connect/ at the reference commit |
| Core expression-syntax provenance | `Appendix I canonical EXPR-* rows; SqlBaseParser.g4 at the reference commit is candidate provenance only` |
| Portable SQL provenance | `Appendix I canonical SQL-* rows; Spark 4.2.0, PostgreSQL 18, DuckDB, and Trino materials are comparison provenance only` |
| Core function provenance | Appendix C canonical FN-\* rows; FunctionRegistry.scala at the reference commit is candidate provenance only |
| Error-registry provenance | common/utils/src/main/resources/error/error-conditions.json at the reference commit |
| Portable configuration | The exact operation and key rows in the canonical bundle |

The reference commit pins source provenance and behavior only for an express delegation satisfying §6.1.1. The separately recorded bundle publication commit and SHA-256 pin interface membership. A hosted-documentation link is informative unless it contains the explicit 4.2.0 version. If generated documentation and the pinned sources, canonical bundle, or this specification disagree, §6.1.1 determines responsibility and the discrepancy MUST be resolved before finalization.

### 22.1.2 No automatic inheritance

A Spark commit or release after the pinned snapshot cannot change Spark Connect 1.0. New RPCs, fields, enum values, relations, expressions, commands, functions, expression productions, providers, configurations, errors, or client APIs remain outside `SC-1.0-P1` until a later specification version publishes a new bundle.\
A later Spark release may implement this frozen profile by passing its TCK. Wire compatibility alone does not add a feature.\
The Portable SQL Core changes only through a new Spark Connect specification profile and bundle. Complete statement-SQL dialect profiles evolve independently. A new Spark SQL grammar or function does not change the Portable SQL Core or an already published Spark SQL dialect profile; it requires a new version of the applicable dialect artifact.

### 22.1.3 Draft and finalization gates

A specification version MAY be drafted before its implementation ships. It MUST NOT become Final until:

1. A named Apache Spark release implements every applicable `SC-1.0-P1` row.\
2. The deterministic core bundle is committed at `docs/spark-connect/spec/manifests/sc-1.0-p1.json` in an immutable Apache Spark publication commit.\
3. §22.1.1 records that publication commit and the lowercase SHA-256 of the complete committed bundle.\
4. The bundle contains reviewed wire, client, provider, function, expression-syntax, portable-SQL, cast, Catalog-schema, deprecation, configuration, error, and Arrow rows.\
5. The `SC-1.0-P1-CLIENT` member has been generated from the approved pinned inputs and contains no required API that needs an excluded node, function, syntax form, or presentation relation.\
6. Every positive and negative core TCK case cites canonical row IDs, relation/expression cases construct protobuf plans directly, and SQL-text cases use only `SC-1.0-P1-PORTABLE-SQL`.\
7. The reference server and clients pass the complete core TCK.\
8. The specification, TCK, bundle, and implementation identify the same profile and immutable pins.\
9. Generated appendices and the informative Google Sheet match the bundle.\
10. The SPIP and PMC review explicitly ratifies both the query-only Portable SQL Core as a new Spark Connect subset with the boundary in Appendix I and the unknown-session `FetchErrorDetails` materialization rule in §7.4.1.\
11. The Required Profile source-validation list required by §22.6 is complete for every in-profile normative rule derived from reference source or tests, wherever that rule is stated. It covers protocol, errors and lifecycle; types, casts, and Arrow; relations; expressions and functions; commands and writes; configuration and Catalog; providers; and clients. It validates accepted values, boundary inclusivity, NULL, NaN, and empty-input behavior, rank and error bounds, tie behavior, invalid request shapes, and analysis-time reads and side effects, including every schema-inference path governed by §9.2.\
12. Every source-validation entry records its normative section, canonical row, pinned source or test path, exact behavior checked, corresponding TCK case, and result. Every difference is corrected in the specification and TCK or recorded as a release-blocking discrepancy.\
13. All other known discrepancies are resolved.\
14. The Apache Spark governance process accepts the specification.

A complete Spark SQL or other statement-dialect profile is a separate artifact with a separate finalization gate. Core finalization requires the Portable SQL Core but does not require any complete engine dialect.\
Any Spark change intended for a future Required Profile needs a specification-impact determination plus coordinated bundle and TCK changes. Editorial corrections update the draft history but do not rewrite a released bundle.

## 22.2 Additive Protocol Evolution — NORMATIVE

Within a major version:

* new RPCs and messages MAY be added;\
* new fields MAY be added with previously unused field numbers;\
* new enum values and `oneof` variants MAY be added;\
* new built-in functions MAY be added only by publishing a new specification minor version whose canonical bundle includes them; a released bundle never inherits them automatically;\
* an existing field number MUST NOT be reused for a different meaning or incompatible type;\
* existing field semantics and RPC signatures MUST NOT change incompatibly; and\
* stable fields and RPCs MUST NOT be removed.

Proto3 receivers preserve or ignore unknown fields according to protobuf compatibility rules. Application code MUST also handle unknown enum values and `oneof` variants safely. Ignoring an unknown field does not permit a server to claim that it executed an unknown relation, expression, command, or request option.

## 22.3 Explicit Extension Points — NORMATIVE

The Apache Spark proto at the reference version does **not** reserve a general vendor range such as fields 800–997. Field numbers are message-local, and downstream implementations MUST NOT treat unreserved numbers in `spark.connect` messages as privately allocated vendor space.\
Extensions use only the explicit `google.protobuf.Any` fields declared by Apache Spark. The core extension points include:

* `UserContext.extensions` (repeated field 999);\
* `ExecutePlanRequest.RequestOption.extension` (field 999);\
* `ExecutePlanResponse.extension` (response-type field 999);\
* `Relation.extension` (field 998);\
* `Expression.extension` (field 999);\
* `Command.extension` (field 999); and\
* request/response and operation-level `GetStatus.extensions` fields (field 999).

The number 998 or 999 is not a protocol-wide reservation by itself; each extension point exists only where the Apache Spark message explicitly declares it.\
An extension payload MUST:

* use a globally unique protobuf type URL;\
* be defined in a package outside `spark.connect` unless accepted into Apache Spark;\
* preserve all standard semantics when absent;\
* fail explicitly when the selected relation/expression/command extension cannot be resolved; and\
* avoid placing secrets in contexts that may be logged or returned without protection.

Unknown ordinary proto fields are handled by proto3 compatibility. An explicitly selected `Any` plan extension is different: if the server cannot unpack or implement it, it MUST return an unsupported-extension error rather than execute a child/no-op approximation.\
Adding a new standard field directly to an Apache Spark message requires the normal Apache Spark contribution and compatibility process. A downstream distribution may carry private proto changes, but those changes are outside Spark Connect v1.0 and have no collision-free guarantee from this specification.

## 22.4 Compatibility Guarantees — NORMATIVE

**Backward compatibility.** A client targeting version `N.M` MUST continue to operate against a server implementing a later compatible minor version `N.M'`, for the v`N.M` surface.\
**Forward handling.** A newer client may send additive fields or enum values unknown to an older server. Proto decoding must remain safe, but the older server is required to execute only the surface it implements. If an unknown selected operation is semantically necessary, the server MUST return `UNIMPLEMENTED` or a structured unsupported-feature error rather than claim success.\
**Cross-major compatibility.** Compatibility across different major versions is not guaranteed.\
These guarantees apply to the standard Apache Spark surface. Extension interoperability requires both endpoints to understand the same payload type and semantics.

## 22.5 Deprecation Policy

A standard field, message, RPC, enum value, or function may be deprecated in a minor version. A deprecated entity:

* remains decodable and functional for the compatibility period declared by Apache Spark and this specification;\
* is marked `[deprecated = true]` in proto where applicable, or clearly identified in this document;\
* is listed in the revision history; and\
* is removed only through an explicitly approved incompatible evolution process.

Clients SHOULD stop emitting deprecated fields when a supported replacement exists. Servers MUST continue to accept the deprecated form for the required compatibility period and MUST define conflicts when old and new forms are both supplied.

### 22.5.1 Deprecation manifest

`SC-1.0-P1-DEPRECATIONS` is the machine-readable deprecation inventory for the Required Profile. Each row records a stable row ID, entity kind, canonical entity ID, version introduced, version deprecated, replacement when any, last version in which support is required, earliest permitted removal version, and applicable TCK phase.\
An empty deprecation array means that no entity in this Required Profile is deprecated. It does not erase deprecations for optional or out-of-profile Spark APIs. Prose, proto annotations, or source annotations that disagree with a canonical deprecation row are a release-blocking defect; they do not silently change the row.\
A later minor specification may add a deprecation row while retaining the entity. Actual removal requires the incompatible evolution process described above and a new major version unless an already-published compatibility rule expressly permits otherwise.

## 22.6 Reference-Source Discipline

Every protocol field and field number quoted by this specification MUST be traceable to the Apache Spark files under `sql/connect/common/src/main/protobuf/` at the pinned reference commit. Databricks Runtime or other downstream-only fields MUST NOT be included in the standard surface.\
Reference-source code and reference-server tests are validation evidence; they do not create a portable rule. A behavioral requirement MUST be stated directly in normative prose, be a constraint named by a canonical row, or be an express delegation satisfying §6.1.1. An express delegation MUST identify the pinned reference version, the exact behavioral question, and its controlling canonical row or normative section. “Testable against Spark,” an uncited source path, or an observed implementation result is not a delegation and cannot fill a normative gap.\
A future edit that depends on a downstream fork MUST label that material as a non-standard extension and keep it outside the conformance profile. The finalization audit MUST search normative prose for implementation-dependent phrases and resolve each one as direct portable text, an express bounded delegation, or an out-of-profile statement.\
Before Final status, the specification project MUST publish a Required Profile source-validation list. The list MUST cover every in-profile normative rule derived from pinned reference source or tests, regardless of chapter or appendix. At minimum, its audit scope includes protocol, error, and lifecycle behavior; types, Arrow mappings, and casts; relations; expressions and functions; commands and writes; configuration and Catalog; provider behavior; and client lowering and validation.\
Each entry MUST identify the normative section and canonical row, the pinned source or test path used as evidence, the exact boundary or behavior checked, the corresponding TCK case, and the result. The checks MUST include, where applicable, accepted values, boundary inclusivity, NULL and NaN handling, empty-input behavior, rank or error bounds, tie behavior, invalid request shapes, and analysis-time reads and side effects. For each schema-inference read, the entry MUST state whether it reads source data, source metadata, or an input relation and record the applicable sampling, discovery, and format controls. The invalid-shape audit MUST cover relations, expressions, and commands and record whether the portable contract requires client rejection or non-emission, server rejection, or expressly leaves the server outcome outside conformance. Implementation permissiveness alone cannot create another exception. This list is validation evidence only; it cannot replace normative prose, a canonical row, or a TCK assertion.
