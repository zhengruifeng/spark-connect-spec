# 7. Errors

This chapter defines the canonical Spark Connect failure transport. A failed unary RPC or a failed server-streaming RPC terminates with a non-`OK` gRPC status. It does **not** return an application error inside its normal protobuf response. In particular, `ExecutePlanResponse` has no error field.\
For a streaming RPC, valid responses may precede the terminal gRPC error. Those responses form a partial failed result and MUST NOT be presented as successful completion. No normal response follows a terminal status.

## 7.1 Canonical gRPC status-details envelope

A Spark application failure is encoded as `google.rpc.Status`, transported through the standard gRPC status-details mechanism (`grpc-status-details-bin` in implementations that expose the header directly):\
message google.rpc.Status {\
  int32 code \= 1;\
  string message \= 2;\
  repeated google.protobuf.Any details \= 3;\
}\
A conforming server reporting a structured Spark application failure places a packed google.rpc.ErrorInfo in details. A conforming client MUST inspect status details before falling back to the plain status message. A server MUST NOT invent an error\_id field in ExecutePlanResponse, AnalyzePlanResponse, or another ordinary response.\
The portable ErrorInfo contract is:

* `domain` is `org.apache.spark`.\
* `reason` identifies the server exception type and is diagnostic; portable control flow uses `errorClass` and `sqlState` metadata.\
* metadata key `classes`, when present, is the JSON exception-class hierarchy and is diagnostic.\
* metadata key `errorClass`, when present, is the `SparkThrowable` condition from the frozen v1.0 error registry.\
* metadata key messageParameters, when present, is a JSON object whose keys follow that registry entry. The server MAY omit it when the serialized metadata would exceed its configured gRPC metadata limit.\
* metadata key `sqlState`, when present, is the corresponding SQLSTATE.\
* metadata key `errorId`, when present, is an opaque lookup token for §7.4. It is not present unless enrichment was enabled and the server retained the throwable in the identified session.\
* metadata key `stackTrace`, when present, is diagnostic and configuration-dependent.

The plain `Status.message` may be abbreviated. Clients SHOULD prefer structured metadata and successfully fetched details, but MUST remain functional when only the code and message are available.\
Unless a canonical error row assigns another standard status, an ordinary non-fatal application error uses gRPC code INTERNAL plus ErrorInfo. The error class does not by itself imply a different gRPC code.

## 7.2 gRPC status codes

The protocol does not define a closed list of possible gRPC codes. The gRPC runtime, intermediaries, deployment security layer, resource limits, deadlines, cancellation, and the Spark Connect service may all terminate a call. Clients MUST safely handle every standard code. Authentication and authorization code generation is outside conformance; common cases include:

| Code | Meaning in Spark Connect |
| :---- | :---- |
| `CANCELLED` | The client, server, or an explicit interruption cancelled the call. |
| `INVALID_ARGUMENT` | The request could not be accepted because its envelope or transport-level input was invalid. |
| `DEADLINE_EXCEEDED` | A client or intermediary deadline expired; execution completion may be indeterminate. |
| `NOT_FOUND` | A referenced retained resource was not found when that condition is represented at transport level. |
| `RESOURCE_EXHAUSTED` | A gRPC message limit, memory/resource quota, or server capacity limit was exceeded. |
| `UNAUTHENTICATED` | Authentication failed in a deployment-specific layer; the TCK does not require this code to be generated. |
| `PERMISSION_DENIED` | A deployment-specific access-control layer denied the request; the TCK does not require this code to be generated. |
| `UNAVAILABLE` | The endpoint or an intermediary is temporarily unavailable. |
| `UNIMPLEMENTED` | An OPTIONAL or DEFERRED RPC is not implemented. It is invalid for an RPC required by `SC-1.0-P1`. |
| `INTERNAL` | The portable mapping for ordinary application errors and unexpected server failures. Inspect ErrorInfo. |
| `UNKNOWN` | The failure could not be mapped more specifically. |

`OK` denotes successful RPC termination and never carries an application failure. A client MUST NOT automatically retry solely from the code; Chapter 10 controls retry and side-effect hazards.

## 7.3 `SparkThrowable` metadata

When the underlying exception implements `SparkThrowable`, the `ErrorInfo` metadata may contain:

* `errorClass`: stable identifier from the frozen v1.0 registry when it fits within transport metadata limits;\
* `messageParameters`: the registry-defined parameter map, serialized as JSON when it fits;\
* `sqlState`: the registry-defined SQLSTATE when one exists.

The v1.0 registry snapshot fixes identifiers, SQLSTATE values, and parameter names; it does not require every implementation to use the same JVM exception class. Missing optional metadata does not convert a failed status into success.

## 7.4 Enriched details and `FetchErrorDetails`

`FetchErrorDetails` is a bounded, best-effort, at-most-once enrichment mechanism. The original gRPC status and `ErrorInfo` are the canonical failure record and MUST remain sufficient for a client to surface the failure.\
A client may attempt `FetchErrorDetails` only when all of these preconditions hold:

1. the terminal status contains a packed `ErrorInfo`;\
2. that `ErrorInfo` contains a non-empty `errorId` metadata value;\
3. the request uses the same `session_id` as the failed call;\
4. `client_observed_server_side_session_id`, when supplied, identifies the current session realization; and\
5. the client has not consumed or abandoned that error token.

The token is opaque and session-scoped. Authentication and authorization checks before lookup are deployment-specific and outside conformance.\
The reference implementation creates `errorId` only when `spark.sql.connect.enrichError.enabled=true` and a live session holder is available. It retains at most 20 errors per session, expires entries after bounded inactivity, and removes an entry when the first fetch begins. Therefore size pressure may evict an entry before its nominal timeout, only one concurrent fetch may receive the details, a lost response cannot be fetched again, session release or loss removes the details, and enrichment disabled before session lookup produces no token.\
The specification imposes no minimum retention duration and no durable or idempotent retrieval guarantee. A missing, expired, consumed, or unknown token returns a `FetchErrorDetailsResponse` without `root_error_idx`. The client then falls back to the original `ErrorInfo` and status message; it MUST NOT report success or replace the original failure with a generic lookup failure.

### 7.4.1 Unknown-session materialization: design decision requiring ratification

The protobuf IDL identifies the logical session but does not require an error-detail lookup to create session state. Apache Spark 4.2.0 nevertheless calls `getOrCreateIsolatedSession` before reading the error cache. The proposed `SC-1.0-P1` rule deliberately preserves that observable reference behavior: `FetchErrorDetails` for an unknown, non-tombstoned session creates a fresh session and returns an empty-details response. A following existing-session lookup can observe that new realization as specified in §8.4.1.\
Creation is not necessary to return an empty lookup result. An existing-only lookup that creates no state is a reasonable alternative for another engine, but it would not match the pinned reference server. The SPIP and PMC review MUST explicitly ratify the reference-compatible rule before this specification becomes Final. If it is not ratified, the profile, session matrix, and TCK MUST be revised together; silence does not select either behavior.

### 7.4.2 Fetched error graph

On a successful fetch, `root_error_idx` selects the top-level entry in `errors`. Each `Error.cause_idx`, when present, indexes another entry in the same response. Every index MUST be in bounds and the cause graph MUST be acyclic.

## 7.5 `QueryContext`

The fetched `SparkThrowable.query_contexts` list identifies source locations in order.\
For SQL contexts, `object_type` and `object_name` identify the relevant object when available; `start_index`, `stop_index`, and `fragment` identify the source span.\
For DataFrame contexts, fragment describes the DataFrame operation and call\_site identifies client user code when available. Context is diagnostic and has no standardized security meaning.

## 7.6 Stack traces and diagnostic fields

`Error.error_type_hierarchy`, `Error.stack_trace`, `ErrorInfo.reason`, `classes`, and `stackTrace` are diagnostic. Class names and stack frames are implementation details and MAY be absent. Clients MUST NOT use them for conformance-relevant control flow.\
Server stack traces are controlled by the pinned configuration profile and may be omitted for security, size, or policy reasons. Secrets MUST be redacted.

## 7.7 Error TCK requirements

The TCK MUST verify terminal gRPC failure rather than an in-band response, decoding of google.rpc.Status and packed ErrorInfo, fallback when errorId is absent, enrichment-enabled successful fetch, enrichment-disabled behavior, same-session enforcement, single-consumer behavior, cancellation, resource exhaustion, partial-stream failure, and malformed cause-index rejection. It does not test authentication or authorization. It MUST NOT require 60 seconds of guaranteed retention or retryable enriched-detail retrieval.
