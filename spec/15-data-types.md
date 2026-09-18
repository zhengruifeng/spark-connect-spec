# 15. Data Types

`DataType` describes a logical Spark SQL type. It does not carry a value. The selected `kind` variant and its nested fields define the complete type used in schemas, casts, UDF signatures where supported, and typed null literals.

## 15.1 Common type rules

Exactly one `DataType.kind` variant MUST be selected. A missing kind, invalid required nested field, or unsupported selected kind is an error. The receiver MUST NOT guess a type from unrelated populated fields.\
`type_variation_reference` fields are part of the wire message but no nonzero portable semantics are assigned by v1.0. A v1 implementation MUST preserve compatible unknown data and MUST NOT use a private interpretation to change the meaning of a required standard type.

## 15.2 v1.0 required type profile

Required membership is fixed exclusively by manifest SC-1.0-P1 (§6.3). The following table explains the manifest-selected kinds:

| Category | Required kinds |
| :---- | :---- |
| Null | `NULL` |
| Integral | `Byte`, `Short`, `Integer`, `Long` |
| Fractional | `Float`, `Double`, `Decimal` |
| Other primitive | `Boolean`, `String`, `Binary` |
| Date/time | `Date`, `Timestamp`, `TimestampNTZ` |
| Complex | `Array`, `Map`, `Struct` |

Char, VarChar, interval types, Variant, UDT, Unparsed, Time, geospatial types, and nanosecond-capable timestamp variants are valid proto surface but outside SC-1.0-P1. No required operation or core TCK case may depend on one of these types.\
A server MUST NOT advertise a required operation and then coerce one of its required result types to string because the type is inconvenient to implement.

## 15.3 Primitive types

### 15.3.1 Integral types

`Byte`, `Short`, `Integer`, and `Long` are signed two's-complement logical integers of 8, 16, 32, and 64 bits. The canonical operator and CAST-\* rows define arithmetic overflow and cast outcomes for each evaluation mode and spark.sql.ansi.enabled value. No host-language overflow rule applies implicitly.

### 15.3.2 Floating-point types

`Float` and `Double` use IEEE 754 single and double precision. Implementations MUST preserve signed zero and infinities. Every NaN compares equal to every NaN for equality, sorts after all non-NaN numeric values in ascending order, forms one grouping key, and hashes consistently with that equality. A server MUST NOT substitute different host-language NaN rules.

### 15.3.3 `Decimal`

`Decimal carries optional precision and scale. When both are omitted, the logical type is Decimal(10,0). When one is present, both MUST be present. Precision must be from 1 through 38 and scale from 0 through precision. The canonical CAST-* and decimal-operator rows define overflow for ANSI, legacy, and try evaluation; no mode permits silent digit truncation outside those rows.`

### 15.3.4 `Boolean`, `String`, and `Binary`

`Boolean` has true and false values; null remains distinct. `String` is Spark UTF-8 text. The empty string is not null. `Binary` is an arbitrary byte sequence with no text encoding.\
The v1.0 required collation sub-profile contains exactly UTF8\_BINARY. A required String with no explicit collation and one explicitly marked UTF8\_BINARY compare by unsigned UTF-8 bytes; equality, ordering, grouping, and hashing MUST agree with that byte identity. Required function behavior comes only from its FN-\* row. Implementations MUST preserve the collation through schemas, expressions, Arrow round trips, and nested types.\
Collation values other than UTF8\_BINARY, including UTF8\_LCASE, UNICODE, and their variants, are outside the v1.0 Required Profile. A server MAY implement them as extensions. When it does not, it MUST reject an explicit non-default collation with a structured analysis or unsupported-feature error. It MUST NOT erase the annotation, substitute UTF8\_BINARY, or execute with host-language string rules.\
collate and collation are not core functions. The data-type TCK still covers implicit/default and explicit UTF8\_BINARY schema and Arrow round trips, plus a non-default-name negative case that verifies explicit rejection rather than binary fallback.

## 15.4 Date and time types

`Date` is a calendar date encoded in literals as days from the Unix epoch. It has no time-of-day or time zone.\
`Timestamp represents an instant in microseconds from the Unix epoch. Any required conversion between Timestamp and calendar fields or text uses spark.sql.session.timeZone unless an explicitly required provider option overrides it; changing the zone changes rendering, not the instant.`\
`TimestampNTZ` represents a local date-time without time-zone conversion, also carried in microsecond units. It is a distinct logical type from `Timestamp` even when physical fields look similar.\
OPTIONAL `Time` carries fractional-second precision. OPTIONAL `TimestampNTZNanos` and `TimestampLTZNanos` support precision 7, 8, or 9 and remain distinct NTZ/LTZ kinds. Invalid precision MUST be rejected.

## 15.5 Complex types

### 15.5.1 `Array`

`element_type` is required and `contains_null` states whether an element may be null. Array order is semantic. A null array is distinct from an empty array and from an array containing null.

### 15.5.2 `Map`

`key_type` and `value_type` are required. Keys are non-null. `value_contains_null` controls values independently of the map's own nullability.\
The Required Profile uses duplicate-map-key policy EXCEPTION: construction of a map with equal keys fails with the canonical duplicate-key condition. A deployment-only LAST\_WIN mode is outside conformance. A server MUST NOT silently select an occurrence during a core case.

### 15.5.3 `Struct`

`Struct.fields` is ordered. Each field carries `name`, required `data_type`, `nullable`, and optional JSON metadata. Duplicate names may be valid in a schema and MUST be preserved positionally. Metadata absence is distinct from the JSON string for an empty object.

## 15.6 Character and interval types (OPTIONAL)

`Char(length) and VarChar(length) require a positive length. They are outside SC-1.0-P1, and this specification assigns no portable padding, truncation, comparison, or Arrow requirement to them. Appendix I VARCHAR is a Portable SQL grammar alias for required String with UTF8_BINARY, not optional VarChar(length).`\
`CalendarInterval` carries months, days, and microseconds as separate components in literals. `YearMonthInterval` and `DayTimeInterval` may restrict start and end fields. A server implementing them MUST preserve the qualified interval type, not collapse all intervals into a duration.

## 15.7 Variant, geospatial, and UDT types (OPTIONAL)

`Variant is outside SC-1.0-P1. Its value model and Arrow representation require a separately named profile; core conformance neither infers nor tests them.`\
Geometry and Geography carry an SRID and are distinct logical types, but both are outside SC-1.0-P1. This specification defines no portable byte/text conversion for them. An extension profile that selects either type MUST define its physical encoding, SRID domain, validation failures, casts, and Arrow mapping.\
`UDT` carries type/class metadata and an optional underlying `sql_type`. A receiver that does not support the UDT MUST reject it or operate through a specifically permitted underlying-type fallback. It MUST NOT instantiate arbitrary classes named by an untrusted client without a trusted registration mechanism. UDT is a data-type extension, not a UDF execution mechanism. UDT remains outside the required v1.0 profile and TCK; a conforming server MAY reject it. Python UDT metadata may carry an underlying sql\_type, while JVM class loading and artifact distribution are not portable v1.0 contracts.

## 15.8 `Unparsed` types

`Unparsed is outside SC-1.0-P1. A server may parse data_type_string under a separately named type-syntax profile; otherwise it returns a structured unsupported-type error. It MUST NOT reinterpret arbitrary text as a vendor type during a core case.`

## 15.9 Literal correspondence

`Expression.Literal.literal_type` carries values, while `Literal.data_type` carries type information required for null and nontrivial complex values. A typed null MUST select the null literal and provide its logical type. It is distinct from an absent expression.\
Literal units are normative: date is days from the epoch; standard timestamps are microseconds; optional Time is nanoseconds since local midnight in the range 0 through 86,399,999,999,999; optional nanosecond timestamps use epoch microseconds plus nanos\_within\_micro in the range 0 through 999.\
Map literal key and value counts MUST agree. Struct literal element count and order MUST agree with its `Struct` type. A complex literal whose value and declared type disagree is invalid.

## 15.10 Coercion, equality, and casts

Type coercion is performed by server analysis. For required function arguments, comparisons, unions, aggregates, assignment, and complex-type merging, only the applicable FN-\*, REL-\*, and CAST-\* rows permit a coercion. An absent coercion row requires a structured analysis error. Clients MUST NOT pre-coerce solely according to host-language rules.\
Cast behavior is defined by Chapter 17, Appendix B, and the canonical CAST-\* rows. Appendix I type names map to required logical types before cast analysis; VARCHAR maps to String with UTF8\_BINARY and TIMESTAMP maps to Timestamp regardless of spark.sql.timestampType. ANSI, legacy, and try evaluation remain distinct. Type equality MUST retain decimal precision and scale, timestamp flavor, collation, character length, interval qualifiers, SRID, and nested element, value, and field nullability where the selected type carries them.

## 15.11 Arrow mapping and unsupported types

Required result types MUST map losslessly according to canonical SC-1.0-P1-ARROW rows; Appendix B is their informative generated view. Proto and Arrow are two representations of the same logical value.\
For an OPTIONAL unsupported type, the server MUST return a structured unsupported-type error. It MUST NOT map it to `String`, `Binary`, or `NULL`. Older clients encountering a new kind MUST treat it as unsupported rather than interpreting the default enum/oneof state as `NULL`.

## 15.12 Conformance cases

The TCK MUST include numeric boundaries and overflow, NaN and infinities, decimal precision/scale boundaries, empty versus null values, timestamp/time-zone transitions, nested nullability, duplicate struct names, map duplicate-key policy, invalid complex literals, and round trips through both proto literals and Arrow results.
