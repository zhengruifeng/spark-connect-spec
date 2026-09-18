# G. Client API Membership Manifest

`SC-1.0-P1-CLIENT` is the exhaustive client API membership manifest for Python and Scala. This appendix does not define membership. The normative rows are the `client.overloads`, `argument_constraints`, and `unsupported_cases` members of the canonical profile bundle defined by §6.3 and Appendix J.\
The SC-1.0-P1-CLIENT Google Sheet is an informative source-inventory view for review and navigation. Its currently displayed snapshot, labeled 1.0-draft-v0.14, contains 1,409 source-inventory rows in its Manifest tab. It has not been regenerated from a published canonical SC-1.0-P1 bundle and is neither membership nor publication evidence. The unchanged UTF-8 RFC 4180 CSV SHA-256 99ae77f9e98640e47807fa91e3a68fa13221cbd828586c23f3d786c55317ce2a covers only that unchanged Manifest tab; it is not the normative profile digest.\
Every canonical overload row identifies exactly one fully qualified public symbol overload and references canonical constraint IDs. Source declarations, source line numbers, this appendix's family index, the Google Sheet, and generated API documentation are provenance or navigation aids only. They cannot add an alias, overload, default, or argument form. A source declaration with no `CLI-*` row in the canonical bundle is outside client conformance.\
The controlling source inputs were the public Python declarations under `python/pyspark/sql/` (including Connect session provenance where the shared facade lacks the member) and the Scala declarations under `sql/api/src/main/scala/org/apache/spark/sql/`, all at the pinned reference commit. Generation separately records every decorated Python overload and every selected Scala overload, then filters excluded UDF, worker, typed-Dataset, optional-type, provider, and SQL domains.

| Column | Meaning |
| :---- | :---- |
| `row_id` | Stable `CLI-*` identifier cited by TCK cases. |
| `language` | `python` or `scala`. |
| `family_row_id` | Diagnostic `PY-*` or `SC-*` family used by the index below; it is not a wildcard. |
| `fq_symbol` | Fully qualified public symbol, including builder/action types where applicable. |
| `signature` | One exact pinned overload signature, including defaults and return shape/type. |
| `source_path`, `source_line` | Pinned provenance for audit; never open-ended incorporation by reference. |
| `constraint_ids` | Referenced ARG-\* constraints in the canonical bundle; G.3 is an informative summary. |
| `status` | Informative source-inventory status. Before finalization, the Sheet must be regenerated from the immutable SC-1.0-P1-CLIENT member identified by the recorded publication commit and whole-bundle SHA-256. |

The informative Sheet mirror also contains Metadata, Constraints, and Unsupported Cases tabs. Its CSV digest covers only the Manifest header and data rows; Sheet content and display formatting are excluded from normative identity.

## G.2 Family index (informative)

The following index helps readers navigate the machine-readable rows. It does not define overload membership; each named family expands only to the `CLI-*` rows whose `family_row_id` equals that value.

| Language | Family ID | Indexed surface |
| :---- | :---- | :---- |
| Python | `PY-SESSION-1` | Session discovery, lifecycle, identity, configuration/catalog/read properties. |
| Python | `PY-SESSION-2` | `table`, `sql`, `range`, and local `createDataFrame` entry points. |
| Python | `PY-SESSION-3` | Interrupt operations. |
| Python | `PY-CONF-1` | Required `RuntimeConfig` operations and keys. |
| Python | `PY-CATALOG-1`, `PY-CATALOG-2` | Required catalog database, catalog, table, column, function, and view operations. |
| Python | `PY-READ-1` | Required reader builders, formats, paths, options, schemas, and tables. |
| Python | `PY-DF-REL-1`–`PY-DF-REL-4` | Required DataFrame relational transforms. |
| Python | `PY-DF-ACTION-1` | Finite-batch actions whose required lowering stays within core wire nodes; presentation show rows require regeneration as optional. |
| Python | `PY-DF-ANALYZE-1` | Required schema and analysis operations. |
| Python | `PY-DF-VIEW-1` | Required temporary/global view creation. |
| Python | `PY-DF-NASTAT-1` | Required NA and statistical operations. |
| Python | `PY-WRITE-1`, `PY-WRITE-2` | Required V1 and V2 writer operations. |
| Python | `PY-MERGE-1` | Merge writer and matched/not-matched action builders. |
| Python | `PY-EXPR-1` | Required `Column` operators and expression methods. |
| Python | `PY-FUNC-1` | Same-named public function wrappers explicitly enumerated by `CLI-*` rows and limited by Appendix C. |
| Scala | `SC-SESSION-1`–`SC-SESSION-3` | Session lifecycle, SQL/local-data/range entry points, and interrupt operations. |
| Scala | `SC-CONF-1` | Required `RuntimeConfig` operations and keys. |
| Scala | `SC-CATALOG-1` | Required catalog operations. |
| Scala | `SC-READ-1` | Required reader builders and overloads. |
| Scala | `SC-DF-REL-1` | Required `Dataset[Row]` relational overloads. |
| Scala | `SC-DF-ACTION-1` | Finite-batch actions whose required lowering stays within core wire nodes; show overloads require regeneration as optional. |
| Scala | `SC-DF-ANALYZE-1` | Required schema and analysis operations. |
| Scala | `SC-DF-VIEW-1` | Required view creation. |
| Scala | `SC-DF-NASTAT-1` | Required Dataset, `DataFrameNaFunctions`, and `DataFrameStatFunctions` overloads. |
| Scala | `SC-WRITE-1`, `SC-WRITE-2` | Required V1 and V2 writer overloads. |
| Scala | `SC-MERGE-1` | Merge writer and public action-builder overloads. |
| Scala | `SC-EXPR-1` | Required `Column` operators and expression methods. |
| Scala | `SC-FUNC-1` | Same-named public function wrappers explicitly enumerated by `CLI-*` rows and limited by Appendix C. |

## G.3 Parameter constraints

| Constraint ID | Applies to | Required domain |
| :---- | :---- | :---- |
| `ARG-SQL` | Python/Scala `SparkSession.sql` | `SparkSession.sql` is a required transport API. Core rows require Appendix I Portable SQL Core text. Named or positional arguments are scalar typed expressions allowed by I.6; the client preserves text and binds structurally. Text outside the core belongs to optional deployment/dialect behavior. |
| `ARG-LOCAL` | `createDataFrame` | Finite in-process rows with an explicit required schema, or rows for which the pinned client infers only required types. Other input-container and runtime forms are outside this client row. |
| `ARG-COLUMN` | DataFrame, Column, and function rows | A column argument is a same-session `Column`, or a string name/ordinal only where the exact canonical overload row admits it. |
| `ARG-JOIN` | Join and set rows | Inputs share `session_id`; join type is one of the pinned required aliases for inner, cross, full, left, right, semi, or anti joins. |
| `ARG-RANGE` | `range` | Signed 64-bit start/end/step; step is non-zero; optional partition count is positive. |
| `ARG-SAMPLE` | `sample`, `randomSplit`, `sampleBy` | Fractions and weights are finite and non-negative; seeds fit signed 64-bit range; `sampleBy` keys use required scalar types. |
| `ARG-PROVIDER` | Reader and writer rows | Provider, options, schema, paths, save modes, and defaults are exactly Appendix H. |
| `ARG-TYPE` | Cast, schema, and function rows | Only data types in the required SC-1.0-P1-WIRE row. Optional interval, Variant, UDT, Time, geospatial, char/varchar, and nanos-timestamp wire types are outside the required domain. Appendix I VARCHAR is a Portable SQL alias for required String with UTF8\_BINARY; it does not admit the optional VarChar wire type. |
| `ARG-UDF` | All rows | Worker-backed functions, serialized closures, UDF/UDTF handles, and typed Dataset functions are deferred from v1.0. |
| `ARG-COLLATION` | `collate` and `collation` wrapper rows | collate and collation are outside the core Appendix C function set. A separate profile that adds them defines accepted names, results, and errors. |

The canonical SC-1.0-P1 bundle MUST define:

* `ARG-FUNCTION-KERNEL`: core `Column` and function rows use only Appendix C names and overloads; unlisted wrappers are optional or dialect-profile APIs.\
* `ARG-EXPRESSION-STRING`: expression text uses only Appendix I literals, attributes, operators, null tests, `CASE`, primitive casts, required functions, and `transform` lambdas.\
* `ARG-PORTABLE-SQL`: statement text uses only canonical SQL-\* rows and the fixed Appendix I type-alias mappings; current named/positional argument fields are mutually exclusive, and full-dialect text is outside core client cases.

## G.4 Expected unsupported cases

| Case | Required behavior |
| :---- | :---- |
| Locally detectable excluded argument | Fail before RPC with the pinned language's invalid-argument or unsupported-operation exception. |
| `SparkSession.sql` text outside the Portable SQL Core | The client may send text without parsing it as optional behavior; the server may accept it under a complete dialect or return a structured error. The client MUST NOT rewrite it into another statement or treat the result as a core pass. |
| Reader/writer provider outside Appendix H | The client MAY reject locally. A required-only server returns `DATA_SOURCE_NOT_FOUND`; an extension-aware server MAY accept its implemented provider. The client and server MUST NOT silently fall back to another provider. |
| Cross-session DataFrame/Column composition | Fail locally or during analysis with a structured session-mismatch error; never combine plans silently. |
| API or overload absent from the canonical bundle | If it lowers entirely to standard plans and RPCs, it MAY work as an out-of-profile client convenience API without Any. If it sends a nonstandard payload, it is a wire extension and MUST use an explicit Any field. Its presence, absence, success, or failure is not a 1.0 conformance result. |
| Unlisted function, Portable SQL/ExpressionString production, or presentation API | Reject with a structured error or classify it as an optional/dialect-profile API. Never use a UDF, CallFunction, complete statement parser, or presentation relation to manufacture a core pass. |

Every client TCK case MUST cite one `CLI-*` row and every applicable `ARG-*` constraint. Family IDs may be reported for diagnostics, but a family ID never stands in for the overload row.
