# 5. Services and Messages

## 5.1 The `spark.connect` Package

All messages and services defined in this specification reside in the `spark.connect` Protocol Buffers package. The proto source is located in `sql/connect/common/src/main/protobuf/spark/connect/` in the Apache Spark source tree. The package is organized into the following proto files:

* `base.proto` — `SparkConnectService` definition, request/response messages, session and execution lifecycle messages, error messages.\
* `relations.proto` — `Relation` and all `rel_type` variants.\
* `expressions.proto` — `Expression` and all `expr_type` variants.\
* `commands.proto` — `Command` and all variants (DDL, DML, streaming, ML, catalog).\
* `types.proto` — `DataType` and all variants.\
* `catalog.proto` — Catalog operation messages.\
* `ml.proto`, `ml_common.proto` — Machine Learning command and relation messages.\
* `common.proto` — Shared messages (`StorageLevel`, `ResourceInformation`).

Wire-extension payload messages reside in distinct proto packages and are not part of this specification's normative surface. They are carried only through explicit google.protobuf.Any extension fields and follow §22.3.

## 5.2 `SparkConnectService`

The Spark Connect protocol exposes a single gRPC service, `SparkConnectService`. RPCs marked **Required for v1.0** MUST be implemented by a conforming server with the semantics defined in the chapter referenced. RPCs marked **Deferred** are part of the proto surface but not part of v1.0 conformance; a v1.0 server MAY implement them or return `UNIMPLEMENTED`.\
**Required for v1.0:**

* `ExecutePlan` (server-streaming) — execute a logical plan or command. Chapter 10.\
* `AnalyzePlan` (unary) — inspect a plan without executing it. Chapter 9.\
* `Config` (unary) — get, set, unset, or enumerate session configuration. Chapter 11.\
* `Interrupt` (unary) — cancel a running execution or all executions on a session. Chapter 10.\
* `ReleaseSession` (unary) — release a session. Chapter 8.\
* `FetchErrorDetails` (unary) — retrieve structured details for a previously-returned error. Chapter 7.\
* `CloneSession` (unary) — create a new session with state copied from an existing one. Chapter 8.\
* `GetStatus` (unary) — retrieve operation-level status information. Chapter 10.

**Optional in v1.0 (outside Required Profile SC-1.0-P1):**

* `ReattachExecute` (server-streaming) — resume consumption of a previously started execution. Chapter 10.\
* `ReleaseExecute` (unary) — release execution-side resources. Chapter 10.

**Deferred (paired with UDFs; not part of v1.0 conformance):**

* `AddArtifacts` (client-streaming) — upload artifacts to a session. Chapter 13.\
* `ArtifactStatus` (unary) — check the status of uploaded artifacts. Chapter 13.

A server that does not implement the Required-for-v1.0 RPCs is not Spark Connect-conformant.\
---
