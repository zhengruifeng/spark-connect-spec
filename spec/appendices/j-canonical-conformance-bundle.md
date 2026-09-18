# J. Canonical Conformance Bundle

## J.1 Publication identity

The sole machine-readable Required Profile artifact is `docs/spark-connect/spec/manifests/sc-1.0-p1.json` in the Apache Spark repository. A normative citation includes the profile ID `SC-1.0-P1`, immutable publication commit, repository path, and SHA-256 of the complete file bytes.\
The Apache Spark reference commit and bundle publication commit serve different purposes. The reference commit pins protocol candidates and expressly delegated behavior. The publication commit pins reviewed membership. Both the publication commit and whole-file digest remain finalization blockers in this draft.\
The Portable SQL Core is embedded in the core bundle because it is required. Complete SQL dialect profiles are separate versioned artifacts with their own IDs and whole-file digests. They MUST NOT silently expand the core bundle.

## J.2 Required top-level members

| Member | Required contents |
| :---- | :---- |
| `schema_version` | Version of the JSON bundle schema |
| `profile` | Profile ID, specification version, draft/final state, and parent profile |
| `reference` | Apache Spark tag, commit, and pinned generation-source paths |
| `wire` | Required and excluded RPCs, operations, messages, plan nodes, data types, configuration keys, session-lifecycle rows, and error-registry pin |
| `client` | Required `CLI-*` rows, referenced `ARG-*` constraints, and unsupported cases |
| `providers` | Provider, capability, option/default, V2 action, and unsupported-provider rows |
| `functions` | The closed operator/function `FN-*` rows from Appendix C |
| `expression_syntax` | Complete restricted ExpressionString grammar productions and canonical EXPR-\* rows from Appendix I |
| `portable_sql` | Complete Portable SQL lexical and grammar productions plus canonical SQL-\* parameter, type-alias, function-reference, semantic, rejection, and TCK rows from Appendix I |
| `casts` | Required CAST-\* source/target/mode rows, including every CAST-SQL-\* alias binding from Appendix I |
| `catalog_result_schemas` | Ordered `CATSCHEMA-*` rows |
| `deprecations` | Deprecation rows; an empty array has the §22.5.1 meaning |
| `arrow_mappings` | Required `ARROW-*` mappings, selectors, nested rules, and TCK cases |

Every row has a stable ID unique within its family. Cross-references resolve inside the same bundle. Validation rejects duplicate IDs, dangling constraints, aliases without overload rows, client rows that require excluded wire nodes or functions, TCK references to missing rows, and any undefined grammar nonterminal or lexical reference.\
The SC-1.0-P1-CLIENT member MUST be generated so its required rows use only required wire nodes and the closed Appendix C/I surface. `SparkSession.sql` rows MUST distinguish Portable SQL Core conformance from optional full-dialect acceptance. Presentation APIs, unlisted function wrappers, and APIs that require excluded nodes are optional or belong to another profile.

## J.3 Deterministic serialization and digest

The committed bundle uses UTF-8 without a byte-order mark, LF line endings, two-space indentation, lexicographically ordered object keys, stable row-ID array ordering unless an ordinal is semantic, no generated timestamps or environment paths, and exactly one trailing LF.\
The recorded SHA-256 covers the complete committed bytes, including metadata, client constraints, unsupported cases, mappings, and final LF. A digest of one CSV, Sheet tab, family, or regenerated view is not the profile digest.

## J.4 Mirrors

Appendices and the Google Sheet are informative generated views. Each mirror SHOULD display the profile ID, reference pin, publication commit, whole-file digest, generator version, and generation time. Formatting and comments are not normative.\
A mirror mismatch is a release-blocking generation defect. It never changes the bundle.

## J.5 Core TCK binding

The core TCK records the same profile ID, publication commit, path, and digest. Every positive and negative test cites canonical row IDs. Relation and expression tests construct protobuf plans directly. ExpressionString tests cite EXPR-\* rows. Portable SQL request, parameter, type-alias, result, and rejection cases cite SQL-\* rows and use no unlisted statement feature. Each portable cast case also cites its CAST-SQL-\* row and asserts the exact logical target type.\
The core TCK contains no complete-dialect or eager SQL-command cases. A complete-dialect TCK binds to a separate dialect artifact and may reuse the core transport/session/result tests.\
The run-specific deployment descriptor, adapter contract, report bindings, lifecycle-row applicability, and conformance-gating rules are defined exclusively in §6.6.1. This appendix adds no separate descriptor semantics.

## J.6 Finalization gate

Section 22.1.3 is the sole complete finalization checklist. This appendix adds no separate publication conditions.
