# 12. Catalog

The Catalog API is carried by `Relation.catalog`. It exposes Spark catalog operations as executable relations and returns tabular results through `ExecutePlan`. The proto marks the Catalog messages unstable; this specification fixes the v1.0 conformance profile without claiming that every proto operation is required.\
Catalog conformance is semantic. An implementation may use any metastore or catalog technology, but required operations MUST resolve identifiers, apply session state, return schemas, and report errors exactly as this chapter and their canonical rows specify. Native catalog defaults do not fill an unspecified portable rule.

## 12.1 Envelope and common rules

`Catalog.cat_type` selects exactly one catalog operation. An absent, unknown, or unsupported selected operation MUST fail explicitly. It MUST NOT return an empty table that is indistinguishable from a valid empty listing.\
Catalog relations execute in the logical session identified by the surrounding ExecutePlanRequest. Resolution uses that session's current catalog, current database/namespace, SQL configuration, temporary objects, and deployment context.\
Read results MUST use the exact canonical result-schema row assigned to the operation. A valid listing with no visible matches is a successful empty result. A single-object getter for an absent object returns its canonical structured not-found condition. An \*Exists operation returns false for ordinary absence but still propagates malformed-identifier and other deployment failures.

## 12.2 Identifier and pattern semantics

Fields such as `table_name`, `function_name`, `view_name`, and `db_name` are identifiers, not SQL fragments. A multipart identifier is one or more parts separated by an unquoted dot. Backticks quote one part; a doubled backtick represents one literal backtick. Quoting preserves embedded dots and whitespace but does not force case-sensitive resolution. Empty parts and unclosed quotes fail analysis. Unquoted and quoted parts resolve under the required session case-sensitivity rule.\
An omitted optional `db_name` means the current database/namespace. It is distinct from an explicitly supplied empty string. Temporary objects are session-scoped. Global temporary views use the logical global\_temp namespace.\
Optional pattern fields match complete object names case-insensitively. Leading and trailing whitespace is ignored; | separates alternatives; \* matches any character sequence; other regular-expression operators retain their regular-expression meaning. An invalid alternative matches nothing. The server MUST NOT reinterpret the pattern as SQL or apply it to a fully qualified name.

## 12.3 v1.0 operation profile

Required catalog-operation membership is fixed exclusively by manifest `SC-1.0-P1` (§6.3). The subsections below define the semantics of those selected operations and MUST NOT expand or contract the manifest.

### 12.3.1 Required session catalog and database operations

| Proto operation | Required contract |
| :---- | :---- |
| `CurrentDatabase` | Return the session's current database/namespace. |
| `SetCurrentDatabase(db_name)` | Change subsequent unqualified resolution in this session only. |
| `ListDatabases(pattern?)` | Return visible matching databases. |
| `GetDatabase(db_name)` | Return one database or not-found. |
| `DatabaseExists(db_name)` | Return a boolean existence result. |
| `CreateDatabase(db_name, if_not_exists, properties)` | Create with the requested existence behavior and properties. |
| `DropDatabase(db_name, if_exists, cascade)` | Apply exact existence and cascade behavior. |
| `CurrentCatalog` | Return the current catalog. |
| `SetCurrentCatalog(catalog_name)` | Change current catalog for this session; invalid targets leave prior state unchanged. |
| `ListCatalogs(pattern?)` | Return visible matching catalogs. |

An implementation exposing only one catalog may return that catalog, but MUST still resolve qualified and unqualified required identifiers consistently. It MUST NOT claim a successful catalog change to a name it does not support.

### 12.3.2 Required table operations

| Proto operation | Required contract |
| :---- | :---- |
| `ListTables(db_name?, pattern?)` | Return visible tables and views, including the temporary flag and object-type fields fixed by the operation's CATSCHEMA-\* rows. |
| `ListColumns(table_name, db_name?)` | Return columns in schema order with the exact names, types, nullability, comments, partition/bucket flags, and other fields assigned by its CATSCHEMA-\* rows. |
| `GetTable(table_name, db_name?)` | Return one resolved table/view or not-found. |
| `TableExists(table_name, db_name?)` | Return ordinary absence as false. |
| `CreateTable` | Honor name, path, source, description, schema, and options. |
| `DropTable(table_name, if_exists, purge)` | Honor object type, existence behavior, and purge semantics. |

`CreateTable.options` keys are case-insensitive as specified by the proto. Options are data-source options, not arbitrary server configuration. A supplied schema MUST be a valid table schema and MUST be preserved according to provider/catalog rules.\
Dropping metadata does not permit deletion outside the provider/catalog contract. purge=true may request provider-owned data deletion only for the resolved table location; it MUST NOT recursively delete an arbitrary supplied or neighboring path.

### 12.3.3 Required function operations

`ListFunctions(db_name?, pattern?)`, `GetFunction(function_name, db_name?)`, and `FunctionExists(function_name, db_name?)` are required. The canonical function-resolution row fixes precedence among built-in, temporary, and permanent functions. Results report the temporary flag and metadata in the assigned CATSCHEMA-\* rows.\
Listing function metadata does not imply that v1.0 requires code-bearing UDF registration. Registration commands remain deferred with Chapters 13 and 18.

### 12.3.4 Required view operations

The Catalog relation provides `ListViews`, `DropView`, `DropTempView`, and `DropGlobalTempView`.

* `ListViews(db_name?, pattern?)` returns visible matching views.\
* `DropView(view_name, if_exists) drops only a view; resolving a table or other object type returns the canonical wrong-object-type condition.`\
* `DropTempView(view_name) and DropGlobalTempView(view_name) use distinct local and global scopes. Each returns true exactly when it removed a view and false for ordinary absence.`

View creation is **not** a `Catalog.cat_type` operation. DataFrame-backed temporary/global view creation uses `CreateDataFrameViewCommand`; permanent SQL views may be created through `SqlCommand`. Chapter 19 defines command execution. This separation is normative for wire compatibility.

## 12.4 Optional v1.0 catalog operations

The following proto operations are OPTIONAL in v1.0 unless another required feature depends on them:

* legacy `CreateExternalTable`;\
* `RecoverPartitions` and `ListPartitions`;\
* `IsCached`, `CacheTable`, `UncacheTable`, and `ClearCache`;\
* `RefreshTable` and `RefreshByPath`;\
* `GetTableProperties` and `GetCreateTableString`;\
* `TruncateTable` and `AnalyzeTable`.

These operations do not acquire core semantics merely because a server implements them. A separately named optional Catalog profile must define fields such as storage\_level, as\_serde, and no\_scan. Without that profile, the server returns UNIMPLEMENTED or the standard unsupported-feature error and MUST NOT acknowledge and discard the request.

## 12.5 Create operation details

`CreateExternalTable is optional and has no core default contract. Required CreateTable additionally carries optional description. Its provider, path, schema, and option defaults come only from the selected provider and Catalog rows; omission is distinct from an explicitly empty string or empty schema.`\
`CreateDatabase.properties is catalog metadata. if_not_exists=true converts only an ordinary already-exists condition into successful no-op; it does not suppress malformed input or other deployment errors.`

## 12.6 Drop and mutation details

`if_exists=true suppresses only ordinary absence. It MUST NOT suppress invalid identifiers, wrong object type, deployment-policy denial, or an unsupported catalog capability.`\
`DropDatabase.cascade=false MUST reject a non-empty database. With cascade=true, it removes the database and the contained objects covered by the canonical Catalog row. Any access-control decision for affected objects is deployment-specific and outside conformance.`\
Successful setters and mutations become visible to later requests according to the catalog's consistency model. The protocol does not add a cross-request transaction. Concurrent external mutations may change results between calls.

## 12.7 Caching and refresh operations

CacheTable, IsCached, UncacheTable, and ClearCache are optional and have no core default or state contract. A separately named cache profile must define StorageLevel defaulting and session visibility; cache state MUST NOT be presented as durable catalog metadata.\
`RefreshTable and RefreshByPath are optional and have no core invalidation contract. A separately named refresh profile must define the affected metadata and data caches. Neither operation changes deployment-specific access policy.`

## 12.8 Result schemas and ordering

Catalog result schemas are part of the observable contract. Implementations MUST preserve the field names, order, types, nullability, and meanings in the applicable CATSCHEMA-\* rows, including database/catalog descriptions, table type and temporary status, column partitioning metadata, and function class and temporary metadata.\
Listing order is normative only when the operation's canonical row states an ordering guarantee. Otherwise the TCK compares listings as unordered row multisets.

### 12.8.1 Canonical Catalog result-schema rows

A prose reference to an implementation API is not sufficient for final conformance publication. `SC-1.0-P1-CATALOG-SCHEMAS` MUST contain one ordered set of rows for every required Catalog operation that returns tabular data.\
Each `CATSCHEMA-*` row records: the Catalog operation row ID, zero-based field ordinal, field name, required Spark SQL type, nullability, semantic meaning, and any ordering guarantee. Nested structures also record child order and nullability. A schema field absent from these rows is not added by a particular metastore or by inspecting a later Spark release.\
The TCK MUST cite the applicable `CATSCHEMA-*` rows and verify exact field order, names, types, nullability, and empty-result schema. Listing rows may still be compared without order when the schema manifest records no ordering guarantee. The finalization gate in §22.1.3 remains unsatisfied until all required Catalog result schemas are present in the canonical bundle.

## 12.9 Deployment policy and information disclosure

Authentication, authorization, principal selection, and catalog visibility policy are outside v1.0 conformance. UserContext has no standardized identity or privilege meaning.\
The TCK prepares objects visible and accessible to its pre-authorized test context; it does not evaluate hidden-object discovery or permission decisions. Implementations define those policies. Errors SHOULD avoid disclosing unrelated hidden objects or credentials in options.

## 12.10 Error and conformance requirements

The server MUST distinguish these in-scope functional outcomes:

* successful empty listing;\
* ordinary non-existence;\
* already exists;\
* malformed or ambiguous identifier;\
* wrong object type; and\
* unsupported operation or capability.

Authentication and authorization failures are deployment-specific and are not conformance distinctions.\
The canonical error row assigned to each in-scope functional failure supplies its error class and SQLSTATE; Chapter 7 supplies the envelope. The TCK MUST cover current-state isolation, omitted versus explicit namespaces, pattern filtering, temporary/permanent object precedence, `if_exists`/`if_not_exists`, wrong-object-type drops, and empty listings. It does not test unauthorized discovery.
