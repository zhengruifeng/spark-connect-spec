# H. Data-Source Provider Profile

`SC-1.0-P1-PROVIDER` is the exhaustive data-source provider manifest. It is derived from the pinned Spark file-source implementations and the v4.2.0 data-source option tables. The corresponding canonical bundle rows control membership; the tables below are readable views and semantic guidance. A connector or option does not become required merely because the reference distribution contains it.

## H.1 Required providers and capabilities

| Row ID | Provider | Required capabilities |
| :---- | :---- | :---- |
| `PRV-PARQUET` | `parquet` | Batch path read/write; schema inference and explicit schema; projection/filter-correct results; partition discovery; V1 save modes and partitioned write; named-table read; all required V2 table actions. |
| `PRV-ORC` | `orc` | Batch path read/write; schema inference and explicit schema; projection/filter-correct results; partition discovery; V1 save modes and partitioned write; named-table read. |
| `PRV-JSON` | `json` | Newline-delimited batch read/write; schema inference and explicit schema; V1 save modes and partitioned write. Multi-line input is required only through the listed option. |
| `PRV-CSV` | `csv` | Batch read/write; explicit schema; string schema by default; optional inference through the listed option; V1 save modes and partitioned write. |
| `PRV-TEXT` | `text` | Batch read/write with one string column named `value`; whole-file mode through the listed option; V1 save modes and partitioned write. |

Correct query results are normative; a particular vectorized reader, filter-pushdown algorithm, file count, partition count, or physical layout is not, except for externally visible partition-directory semantics.\
Provider schema inference occurs during plan analysis and is governed by §9.2. Parquet and ORC inference may read file metadata such as footers. JSON inference may sample or scan data records. CSV inference may read records to determine field count and, when `inferSchema=true`, field types. Each provider MUST honor its canonical format and sampling options and MUST NOT expose result rows or mutate session-scoped state during inference.

## H.2 Common file-source options and defaults

Option keys are case-insensitive. Values sent through Connect are strings after pinned client conversion.

| Row ID | Option/input | Scope | Required behavior/default |
| :---- | :---- | :---- | :---- |
| `OPT-PATH` | `paths` / `path` | read/write | One or more server-accessible paths. Read-path order is preserved where observable. An empty required path list is invalid. |
| `OPT-SCHEMA` | supplied schema | read | DDL or JSON schema limited to required data types. Absent means provider inference except for `text`'s fixed schema and CSV's default string schema. |
| `OPT-GLOB` | `pathGlobFilter` | read | Absent by default; when present, include only matching file names without changing partition discovery. |
| `OPT-RECURSIVE` | `recursiveFileLookup` | read | `false by default. true recursively discovers eligible files below every supplied path and disables discovery of partition columns from directory names.` |
| `OPT-MISSING` | `ignoreMissingFiles` | read | `false` by default. |
| `OPT-CORRUPT` | `ignoreCorruptFiles` | read | `false` by default. |
| `OPT-MODE` | save mode | write | `errorIfExists` by default; `append`, `overwrite`, and `ignore` are required. Matching is case-insensitive. |
| `OPT-PARTITION` | `partitionBy` | write | Empty by default. Named partition columns MUST exist and use required atomic types. |

The TCK supplies a server-readable temporary root and owns every path/table it uses. Shared-filesystem deployment, object-store credentials, URI schemes, encryption/KMS integration, ACLs, and commit-protocol tuning are environmental concerns and are not provider conformance requirements.

## H.3 Provider-specific required options

Only the following options are required. An absent row is outside the provider profile.

| Provider | Required options and defaults |
| :---- | :---- |
| `parquet` | `mergeSchema=false`; `compression=snappy` for writes. Required values for `compression`: `none`/`uncompressed`, `snappy`, `gzip`, `lz4`, `lz4_raw`, `zstd`. |
| `orc` | `mergeSchema=false`; native ORC semantics; `compression=zstd` for writes. Required values: `none`/`uncompressed`, `snappy`, `zlib`, `zstd`, `lz4`. |
| `json` | `multiLine=false`; `mode=PERMISSIVE` with required values `PERMISSIVE`, `DROPMALFORMED`, `FAILFAST`; `primitivesAsString=false`; `prefersDecimal=false`; `allowComments=false`; `allowUnquotedFieldNames=false`; `allowSingleQuotes=true`; `allowNumericLeadingZeros=false`; `allowBackslashEscapingAnyCharacter=false`; `inferTimestamp=false`; `samplingRatio=1.0`; `dropFieldIfAllNull=false`; `dateFormat=yyyy-MM-dd`; `timestampFormat=yyyy-MM-dd'T'HH:mm:ss[.SSS][XXX]`; `timestampNTZFormat=yyyy-MM-dd'T'HH:mm:ss[.SSS]`; `encoding=UTF-8` when writing; read line separators auto-detected from CR/LF/CRLF; write `lineSep=\n`; `compression=none`. Variant-producing `singleVariantColumn` and `explodeEmbeddedArray` are excluded. |
| `csv` | `sep`/`delimiter=,`; `encoding`/`charset=UTF-8`; `quote="`; `escape=\`; `quoteAll=false`; `escapeQuotes=true`; `header=false`; `inferSchema=false`; `preferDate=true`; `enforceSchema=true`; `ignoreLeadingWhiteSpace=false` on read and `true` on write; `ignoreTrailingWhiteSpace=false` on read and `true` on write; empty-string `nullValue`; `nanValue=NaN`; `positiveInf=Inf`; `negativeInf=-Inf`; the same required date/timestamp formats as JSON; `mode=PERMISSIVE` with the same three required values; `multiLine=false`; `compression=none`. `singleVariantColumn` is excluded. |
| `text` | `wholetext=false`; read line separators auto-detected from CR/LF/CRLF; write `lineSep=\n`; `compression=none`. |

The effective timestamp zone for JSON/CSV parsing and formatting is `spark.sql.session.timeZone` unless the required `timeZone` option overrides it with a valid zone ID. Locale-sensitive behavior uses `locale=en-US` unless the required `locale` option supplies another valid BCP 47 tag. Required date/time pattern tokens and parse/format outcomes are enumerated by the provider option rows; no backend-native pattern token is accepted in a core case unless that row lists it.

## H.4 WriteOperationV2 profile

V2 conformance is evaluated with `using("parquet")` against a TCK-owned identifier in the session catalog.

| Row ID | Action | Required behavior |
| :---- | :---- | :---- |
| `V2-CREATE` | `create` | Create a missing table; fail if it exists. |
| `V2-REPLACE` | `replace` | Replace an existing table; fail if it is missing. |
| `V2-CREATE-REPLACE` | `createOrReplace` | Create or replace atomically from the caller's perspective. |
| `V2-APPEND` | `append` | Append rows using by-name table compatibility rules. |
| `V2-OVERWRITE` | `overwrite(condition)` | Replace rows matching the required Boolean expression. |
| `V2-OVERWRITE-PARTITIONS` | `overwritePartitions` | Replace every partition represented by the input and preserve other partitions. |

`partitionedBy` is required for identity transforms on required atomic columns. Bucket, years/months/days/hours transforms, table clustering, distribution/order requirements, third-party catalogs, and provider-defined transaction guarantees are outside v1.0.

## H.5 Unsupported-provider and option behavior

An absent format resolves to `parquet`. Any explicit provider outside H.1 is outside the Required Profile.\
A **required-only server** that does not implement the named provider MUST return the pinned structured `DATA_SOURCE_NOT_FOUND` condition (or a more specific pinned subclass) and MUST NOT retry another provider. An **extension-aware server** MAY accept the same explicit provider when it implements that provider as an extension. Extension success does not satisfy any required-provider TCK row and MUST NOT change calls that use only required providers.\
A client MAY reject an unlisted provider locally. If it sends the name, it MUST preserve that exact provider request and surface the server's result. Neither the client nor the server may silently fall back to `parquet`, another provider, a path interpretation, or an empty relation.\
For a required option, invalid Boolean, numeric, enum, or pattern values MUST fail with a pinned structured analysis error. An implementation MAY accept an option not listed in H.2 or H.3 as an extension, but the TCK MUST NOT rely on that option and the extension MUST NOT change a Required Profile result. Provider extensions MUST NOT change results for calls using only required rows.
