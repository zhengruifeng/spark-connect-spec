# B. Data Type Conversion Tables

This appendix is an informative generated view of the canonical bundle's arrow\_mappings (SC-1.0-P1-ARROW) and casts (SC-1.0-P1-CASTS) members. Canonical ARROW-\* and CAST-\* rows control required membership and are covered by the whole-file digest; Chapters 14, 15, and 17 define portable semantics.

## B.1 Spark Data Types Mapped to Arrow Types

The readable rows below are generated from canonical `ARROW-*` entries. For required `String` and `Binary` mappings, the effective session value of `spark.sql.execution.arrow.useLargeVarTypes` applies recursively through nested types: `false` is the default narrow-width mode; `true` selects the large-variable-width mode.

* `Boolean` → `bool`.\
* `Byte` → `int8`; `Short` → `int16`; `Integer` → `int32`; `Long` → `int64`.\
* `Float` → `float32`; `Double` → `float64`.\
* `Decimal(p, s)` → `decimal128(p, s)`.\
* `String` → `utf8` when `spark.sql.execution.arrow.useLargeVarTypes=false` (the default), or `large_utf8` when it is `true`.\
* OPTIONAL Char(n) and VarChar(n): no core Arrow mapping; a separate profile must define one.\
* `Binary` → `binary` when `spark.sql.execution.arrow.useLargeVarTypes=false` (the default), or `large_binary` when it is `true`.\
* `Date` → `date32[day]`.\
* `Timestamp` → `timestamp[us, <session-time-zone-id>]`. The timezone string is the effective `spark.sql.session.timeZone`; UTC is not hard-coded. Physical values preserve the instant in microseconds.\
* `TimestampNTZ` → `timestamp[us]` with no timezone metadata.\
* OPTIONAL Time: no core Arrow mapping; a separate profile must define one.\
* OPTIONAL `YearMonthInterval` → `interval[year_month]`.\
* OPTIONAL `DayTimeInterval` → `duration[us]`.\
* OPTIONAL `CalendarInterval` → `interval[month_day_nano]`.\
* `Array(t)` → `list(t)` recursively; `Map(k, v)` → `map(k, v)` recursively; `Struct(fields)` → `struct(fields)` preserving order and nullability.\
* OPTIONAL Variant, Geometry, and Geography: no core Arrow mapping; a separate profile must define one.\
* OPTIONAL `UDT` uses the Arrow mapping of its underlying `sql_type`.\
* `Null` → `null`.

## B.2 Arrow Types Mapped to Spark Data Types

Inverse of §B.1: utf8 and large\_utf8 map to String; binary and large\_binary map to Binary. When serializing back to Arrow, the effective spark.sql.execution.arrow.useLargeVarTypes value selects the required width. Arrow extension types not listed above are not recognized.

## B.3 Spark Data Types Mapped to Spark Connect Proto `DataType`

The proto `DataType.kind` oneof has one variant per Spark type. See `types.proto` for the full list.

## B.4 Type Conversions Supported by `Cast`

Each canonical SC-1.0-P1-CAST row expressly delegates only its named source-type, target-type, mode, and failure question to Apache Spark v4.2.0, including the analyzer, Cast implementation, ANSI/session configuration, and the versioned Spark 4.2.0 cast reference.\
For every source/target pair, behavior includes whether the cast is accepted, its result type and nullability, exact versus lossy conversion, overflow behavior, and whether invalid input raises a structured error or returns null. A later Spark cast rule does not enter v1.0 automatically.

### B.4.1 Machine-readable cast matrix

The final `SC-1.0-P1-CASTS` member MUST enumerate every required source-type, target-type, and evaluation-mode combination. Merely pointing to the pinned `Cast` implementation is provenance and is insufficient for an alternative engine to determine conformance.\
Each `CAST-*` row records: source type pattern, target type pattern, applicable mode (`ANSI`, `LEGACY`, or `TRY`), analysis acceptance, result-type and nullability rule, exact or lossy classification, invalid-input behavior, overflow behavior, controlling configuration keys, structured error condition when applicable, and required boundary TCK cases. Complex types record recursive element/key/value/field requirements.\
A pair absent from the canonical matrix is outside the v1.0 required cast profile even if the pinned implementation accepts it through an optional type. Every cast TCK case MUST cite one `CAST-*` row. The canonical bundle generator MUST derive candidates from the pinned Spark snapshot, but only canonical bundle rows define membership.
