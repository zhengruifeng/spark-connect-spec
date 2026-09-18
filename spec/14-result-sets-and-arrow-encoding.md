# 14. Result Sets and Arrow Encoding

This chapter defines tabular result delivery. Spark Connect uses Apache Arrow IPC streaming data inside `ExecutePlanResponse.ArrowBatch`. Chapter 10 defines response identity and operation lifecycle; this chapter defines how clients reconstruct a typed result.

## 14.1 Logical result model

A tabular result consists of one stable Spark `StructType` and an ordered sequence of zero or more logical rows. The transport divides those rows into Arrow batches and may divide a serialized batch into transport chunks. Neither division changes schema, values, or logical row order.\
The service guarantees at least one `ArrowBatch` even when the result contains zero rows. An empty result is therefore represented by a valid Arrow payload whose decoded row count is zero, not by an absent result stream.

## 14.2 `ArrowBatch` fields

| Field | Presence | Contract |
| :---- | :---- | :---- |
| `row_count` | Required | Logical rows in the complete serialized batch. Every chunk of one batch repeats the same value. |
| `data` | Required | Complete Arrow IPC streaming bytes, or a contiguous fragment when chunking is active. |
| `start_offset` | Optional | Zero-based row offset of this batch in the complete execution result. |
| `chunk_index` | Optional | Zero-based fragment index within the serialized batch. |
| `num_chunks_in_batch` | Optional | Total fragments. Presence indicates chunked transport; zero is treated as unchunked for compatibility. |

For an unchunked message, `data` MUST be independently consumable as the serialized Arrow stream produced for that batch. For a chunked batch, individual fragments need not be independently decodable. The client concatenates fragments in `chunk_index` order, then decodes the resulting bytes.

## 14.3 Arrow IPC requirements

The reconstructed data for a batch MUST contain an Apache Arrow IPC streaming-format payload conforming to the Apache Arrow Columnar Format 1.5 specification, with a schema message and zero or more dictionary and record-batch messages followed by the stream end marker. A compatible Arrow reader MUST be able to consume it without implementation-specific preprocessing.\
The Arrow schema of every batch in one logical result MUST be equal. A client detecting a different schema MUST fail the result rather than coerce later batches. A server MUST NOT change field count, field order, names, nested types, or metadata-significant properties between batches.\
Compression, dictionary encoding, buffer alignment, and other Arrow physical choices MAY vary only within the encodings allowed by the canonical SC-1.0-P1-ARROW rows. Clients MUST accept every allowed choice and MUST NOT depend on one buffer layout; servers MUST NOT require an unlisted codec, extension type, or dictionary convention for a Required Profile result.

## 14.4 Schema agreement

`ExecutePlanResponse.schema`, when present, is the Spark Connect logical schema. The schema embedded in every Arrow stream MUST map to that logical schema under Appendix B.\
The response schema is an auxiliary field, not a response-type variant and not necessarily a dedicated response. A conforming client MUST also be able to derive the schema from the first Arrow stream when the explicit field is absent.\
Duplicate Spark field names are legal in contexts where Spark permits them. Clients MUST preserve fields by position and MUST NOT collapse a schema into a name-keyed map. Struct field order, nullability, decimal precision/scale, collation, timestamp flavor, and nested nullability are semantic.

## 14.5 Row count and offsets

After decoding a complete batch, the number of rows MUST equal `row_count`. A mismatch is a protocol error.\
When `start_offset` is present, the first batch normally begins at zero and each later batch begins at the prior batch's start plus row count. Clients SHOULD validate continuity. Missing offsets do not permit reordering; arrival order remains authoritative.\
Offsets count logical rows, not bytes, Arrow messages, or transport chunks. Every chunk of a single batch MUST repeat the same `start_offset`.

## 14.6 Chunking

Chunking is enabled only when the request permits it. For a chunked batch:

1. `num_chunks_in_batch` is the same positive total on every chunk.\
2. `chunk_index` starts at zero and increases without gaps.\
3. `row_count` and `start_offset` are identical on every chunk.\
4. Concatenating each `data` fragment in order reproduces the original serialized Arrow batch exactly.

A client MUST bound buffering according to its resource policy, but MUST assemble all chunks before decoding. Missing, duplicated, out-of-order, or contradictory chunks are protocol failures. A server MUST finish all chunks of one batch before beginning the next batch.

## 14.7 Nulls and nested values

Arrow validity bitmaps represent row-level nulls. They are distinct from schema nullability. A server MUST NOT encode null as a type-specific sentinel such as zero, an empty string, NaN, or an empty collection.\
Arrays preserve element order and contains\_null. Maps preserve key/value association; any NULL map key is invalid. Structs preserve field order. Nested validity at every level MUST agree with the logical type.

## 14.8 Type mapping requirements

Required mapping membership is fixed by SC-1.0-P1-ARROW in the canonical bundle's arrow\_mappings member; Appendix B is an informative generated view. The effective session value of spark.sql.execution.arrow.useLargeVarTypes selects the variable-width encoding: false (the default) maps String to utf8 and Binary to binary, while true maps String to large\_utf8 and Binary to large\_binary. The selected mode applies recursively inside arrays, maps, and structs. ExecutePlan results and client-to-server LocalRelation payloads MUST use the effective session mode consistently. In particular:

* byte, short, integer, and long retain their integer widths;\
* float and double retain their IEEE widths;\
* decimal retains precision and scale;\
* binary is bytes, not UTF-8 text;\
* date uses day units;\
* timestamp and timestamp-without-time-zone remain distinguishable;\
* arrays, maps, and structs map recursively.

An OPTIONAL type without a canonical v1 Arrow row has no core mapping requirement. A server MUST implement it through a separately named profile or reject the result as unsupported; it MUST NOT silently stringify the value.

## 14.9 Completion and partial failure

For reattachable execution, `ResultComplete` has the meaning in Section 10.3.5. It is not part of the Arrow byte stream. A client MUST consume or reattach until it observes authoritative operation completion.\
If the stream ends with an error after one or more batches, those batches form a partial failed result. They MUST NOT be presented as complete. A server MUST NOT emit `ResultComplete` after a terminal error.

## 14.10 Flow control and resource limits

Servers SHOULD honor gRPC flow control and SHOULD avoid unbounded production when the client is slow. Reattachable servers may retain bounded retry state, but retention MUST remain subject to Section 10.6.\
Batch-size configuration is a target/limit used by the implementation, not a guarantee that every logical row can fit below it. Chunking addresses oversized serialized batches at the transport layer. Clients MUST NOT assume a universal batch byte size or row count.

## 14.11 Validation and TCK coverage

Malformed IPC, truncated buffers, schema disagreement, row-count mismatch, impossible offsets, and invalid chunk sequences MUST fail decoding explicitly.\
The TCK MUST cover zero-row results, multiple batches, chunked and unchunked delivery, all-null columns, nested nulls, zero-length binary/string/array values, decimal boundaries, timestamp edge cases, duplicate field names, oversized variable-width values, and failure after partial delivery. It MUST set a non-UTC session time zone and verify both Arrow timezone metadata and instant-preserving round-trip behavior. It MUST run schema and round-trip cases with spark.sql.execution.arrow.useLargeVarTypes=false and true, verifying utf8/binary and large\_utf8/large\_binary respectively for top-level and nested String/Binary values in both ExecutePlan results and LocalRelation input. Optional interval-profile tests MUST verify that DayTimeInterval uses duration\[us\].
