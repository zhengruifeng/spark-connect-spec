# 11. Configuration

The unary Config RPC reads and mutates session-scoped RuntimeConfig. Per §8.4.1, Config materializes an unknown non-tombstoned session. A successful Set or Unset is visible to a later RPC issued after that Config response. An operation captures each relevant setting no later than the start of its parsing or analysis; a concurrent configuration change need not alter work that has already reached that point. Settings MUST NOT leak to another session and MUST NOT survive release of the session.

## 11.1 Request and response envelopes

message ConfigRequest {\
  string session\_id \= 1;\
  optional string client\_observed\_server\_side\_session\_id \= 8;\
  UserContext user\_context \= 2;\
  Operation operation \= 3;\
  optional string client\_type \= 4;\
}\
The server MUST validate session identity before reading or modifying configuration. `client_type` is logging metadata only.\
`operation` selects exactly one of `Set`, `Get`, `GetWithDefault`, `GetOption`, `GetAll`, `Unset`, or `IsModifiable`. An absent or unknown standard operation is an error.\
Every `ConfigResponse` echoes `session_id`, includes the effective `server_side_session_id`, and carries operation-specific `pairs` and `warnings`.

## 11.2 `KeyValue` and absence

`KeyValue.key` is required and `value` is optional. These states are distinct: no pair; a pair without `value`; and a pair with `value = ""`. Clients and servers MUST preserve the distinction.\
Required key names are matched exactly and case-sensitively. Each canonical configuration row defines its accepted value spellings and domain. Invalid values MUST NOT be silently coerced.

## 11.3 `Set`

`Set.pairs` is processed in request order. With `silent` absent or false, the first failure terminates the RPC; pairs applied earlier remain applied because the operation is not transactional.\
With `silent=true`, a failed pair adds a warning and processing continues. Successful pairs remain applied. Silent mode MUST NOT claim that the failed key changed. `Set` returns no result pairs.

## 11.4 `Get`, `GetWithDefault`, and `GetOption`

Get returns the effective value for each requested key. If a key has neither an explicit nor inherited/default value, the RPC fails at that key; use GetOption when absence is expected.\
`GetWithDefault` returns the configured value or the request-supplied default for every pair. Reading with a default MUST NOT install that default.\
`GetOption` returns one pair per requested key. A configured key has a value; an unset key returns a pair whose optional `value` is absent. This distinguishes unset from the empty string.

## 11.5 `GetAll`

Without a prefix, `GetAll` returns all entries exposed by session `RuntimeConfig`; ordering is not semantic.\
With prefix, the server returns only keys beginning with that exact, case-sensitive prefix and removes the prefix from every returned key. Clients MUST NOT apply that transformation when no prefix was requested.\
Servers MUST redact or omit secrets according to their deployment policy. Storing a value in runtime configuration does not require it to be returned by an otherwise permitted read.

## 11.6 `Unset`

`Unset` processes keys in request order and removes their session overrides. A missing key is a no-op. Later reads observe the inherited/default value. The operation returns no result pairs.\
Static or protected keys remain protected; `Unset` MUST NOT bypass server-owned configuration.

## 11.7 `IsModifiable`

The response contains one pair per requested key with lowercase string `"true"` or `"false"`. `true` means runtime-modifiable, not that every string is valid. The operation is observational and MUST NOT mutate state.

## 11.8 v1.0 portable runtime profile

No key list is encoded in base.proto. The canonical SC-1.0-P1-CONFIG rows select the required keys and bind each key's exact spelling, default, accepted value domain, mutability, negative cases, and TCK observations. The controlling portable effects are:

| Key | Required behavior |
| :---- | :---- |
| `spark.sql.session.timeZone` | A session-scoped time-zone identifier used whenever required parsing, casts, Arrow schemas, or rendering convert between an instant and calendar fields. An invalid identifier fails Set. |
| `spark.sql.ansi.enabled` | A session-scoped Boolean controlling the required overflow, division-by-zero, invalid-cast, and related error rows. |
| `spark.sql.legacy.timeParserPolicy` | A session-scoped value of LEGACY, CORRECTED, or EXCEPTION controlling the required date-time parser rows. Other values fail Set. |
| `spark.sql.shuffle.partitions` | A positive integer used as the default partition count by required shuffle operations that do not carry an explicit count. |
| `spark.sql.files.maxPartitionBytes` | A positive byte count used when a required file read plans input partitions. |
| `spark.sql.connect.enrichError.enabled` | A session-scoped Boolean controlling whether a newly reported eligible error may receive an errorId for §7.4. Changing it does not create details for an earlier error. |
| `spark.sql.connect.serverStacktrace.enabled` | A session-scoped Boolean controlling whether structured error metadata may include a server stack trace. It does not change the gRPC code, errorClass, or success/failure outcome. |
| `spark.sql.execution.arrow.useLargeVarTypes` | A session-scoped Boolean: false maps String to utf8 and Binary to binary; true maps them to large\_utf8 and large\_binary. The selected mapping applies recursively to ExecutePlan output and LocalRelation input. |

A conforming server MUST expose every listed key through all Config operations and MUST implement the observable effect selected by its canonical row. It MUST NOT accept a value and then ignore that value. Behavior for an unlisted key is deployment-specific and cannot satisfy a Required Profile case. No unspecified Apache Spark configuration behavior is incorporated by reference.

## 11.9 Static server settings

Apache Spark defines `spark.connect.grpc.arrow.maxBatchSize` and `spark.connect.grpc.maxInboundMessageSize` with `buildStaticConf`. They configure server startup and transport; they are not mutable per-session runtime keys.\
`Config.Set` MUST NOT claim to change either key for one session. If visible through `Config`, `IsModifiable` MUST return false; an implementation may leave them outside `RuntimeConfig`.\
Identity/deployment values such as `spark.app.id`, `spark.app.name`, and `spark.sql.warehouse.dir` may be visible. When exposed as static/read-only values, `IsModifiable` MUST return false.

## 11.10 Deployment policy and sensitive values

It is non-conforming to accept a portable runtime key but ignore its observable effect. A server that cannot honor a value fails `Set`, or warns and leaves it unapplied under `silent=true`.\
Authentication, authorization, and administrative policy are outside v1.0 conformance. Configuration tests run in a pre-authorized environment. Implementations MAY apply additional deployment restrictions, and errors and warnings MUST NOT echo secret values.

## 11.11 Persistence, cloning, and SQL equivalence

Applied settings persist for the session lifetime. They do not retroactively alter an operation that already began parsing or analysis under the prior value.\
A cloned session receives a snapshot; later changes are isolated. Releasing the session releases overrides.\
SQL `SET key = value` and `Config.Set` mutate the same session state. SQL `RESET` restores inherited/default behavior. Both paths MUST be mutually observable through `Config`.

## 11.12 Evolution

Adding a required portable key requires specification and TCK coverage for default, mutability, validation, scope, and observable effect. Removing a required key or changing its semantics incompatibly follows Chapter 22.

## 11.13 Portable warning contract

`ConfigResponse.warnings` is the only portable v1.0 non-fatal warning channel. Each entry is a response-local diagnostic string concerning a deprecated or unsupported configuration examined by that response, or a failed `Set` pair processed under `silent=true`.\
For operations with ordered input keys or pairs, warnings MUST be appended in the same processing order as the inputs that produced them. `GetAll` ordering, including warning ordering, is not semantic because its result ordering is not semantic. A warning is not session state: it is neither chained onto a later response nor retrievable through a separate RPC. Clients MUST NOT parse warning text for portable control flow.\
A successful response containing warnings remains a successful gRPC call, but a warning MUST NOT claim that a rejected `Set` pair was applied. Under `silent=true`, successful earlier and later pairs remain applied as defined by §11.3, while each failed pair produces a warning and remains unapplied. With `silent=false`, the first failure terminates the RPC under Chapter 7 rather than being converted into a warning.\
Server logs, metrics, progress records, and plain text embedded in another response are not portable warnings. Adding a warning channel to another RPC requires a later protocol and specification revision.
