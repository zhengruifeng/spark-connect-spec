# C. Built-in Function Reference

The core function manifest is `SC-1.0-P1-FUNCTIONS`. It is a closed list of `UnresolvedFunction` names and overloads, not a snapshot of an engine's complete registry. Function-name matching is case-insensitive. Every required alias and overload has its own `FN-*` row in the canonical bundle.\
An invocation is required only when a row matches its name, arity, argument type domain, literal/foldability constraints, return rule, and session predicates. An implementation MUST NOT fill a missing row with a UDF, `CallFunction`, an optional data type, or an implementation-specific implicit cast.

## C.1 Operators

| Family | Canonical `UnresolvedFunction` names | Required domain |
| :---- | :---- | :---- |
| Arithmetic | `+`, `-`, `*`, `/`, `%` | Two compatible required numeric values |
| Unary negative | `negative` | One required numeric value |
| Comparison | `==`, `!=`, `<`, `<=`, `>`, `>=` | Two values with a required common comparable type |
| Null-safe equality | `<=>` | Two values with a required common comparable type |
| Boolean | `and`, `or`, `not` | Required Boolean values |
| Null tests | `isNull`, `isNotNull` | One value of any required type |

Arithmetic result types, decimal precision/scale, overflow, division-by-zero behavior, NaN ordering/equality, coercion, and ANSI-mode errors are fixed by the canonical overload and cast rows. Boolean operators use three-valued logic. `<=>` always returns a non-null Boolean.

## C.2 Scalar functions

| Name | Required overloads | Result |
| :---- | :---- | :---- |
| `abs` | One required numeric argument | Same canonical numeric family, subject to overflow rules |
| `coalesce` | One or more arguments with a required common type | First non-null value; common type |
| `nullif` | Two comparable arguments with a required common type | Null when equal, otherwise the first value |
| `lower`, `upper` | One `String` | `String` using the required binary/default collation |
| `length` | One `String` | `Integer` character length |
| `substring`, `substr` | `String, Integer` and `String, Integer, Integer` | `String`; the two names are aliases with separate rows |
| `concat` | One or more `String` arguments | Concatenated `String` |
| `trim` | One `String` | `String` with the canonical space-trim behavior |

These are the only required scalar names. Binary, array, collation-specific, regular-expression, locale-specific, date/time, conversion-function, JSON, Variant, and geospatial overloads are outside the core profile unless another canonical row explicitly adds them.

## C.3 Aggregate functions

| Name | Required overloads | Result |
| :---- | :---- | :---- |
| `count` | `count(*)`, `count(expr)`, and distinct forms over required types | Non-null `Long` |
| `sum` | One required numeric expression, with optional distinct flag | Canonical widened numeric type |
| `avg` | One required numeric expression, with optional distinct flag | Canonical average type |
| `min`, `max` | One expression of a required orderable atomic type | Input/common type |

Null treatment, empty-input results, decimal widening, overflow, distinct behavior, and invalid domains are fixed by the `FN-*` rows. These functions may appear in `Aggregate`, `CollectMetrics`, and `Window` where the enclosing node admits them.

## C.4 Higher-order function

The core requires two `transform` overloads:

* `transform(array<T>, x -> expression<U>) -> array<U>`; and\
* `transform(array<T>, (x, i) -> expression<U>) -> array<U>`, where `i` is a zero-based `Integer`.

`T` and `U` use required data types. The lambda serializes with `LambdaFunction` and `UnresolvedNamedLambdaVariable`. Worker-backed Python/Scala closures are excluded. Array nullability and per-element null behavior follow the canonical rows.

## C.5 Excluded names and safe failure

Every name or overload not listed in C.1–C.4 is outside the core profile, including:

* all user-defined scalar, aggregate, and table functions;\
* ranking and analytic functions such as `row_number`, `rank`, `lag`, and `lead`;\
* collection functions other than the required `transform`;\
* date/time, JSON, Variant, geospatial, random, generator, and provider-specific functions; and\
* multipart names resolved through a server catalog.

An implementation may provide an excluded function as part of its SQL dialect or another extension profile. When it does not, it returns the Chapter 7 envelope with the canonical unresolved-function or unsupported-feature condition. It MUST NOT silently choose a different required function.

## C.6 Canonical rows and TCK

Each `FN-*` row records canonical name, alias identity, positional/named arity, argument type domains, return type and nullability, determinism, configuration dependencies, structured errors, and boundary cases.\
The core TCK cites those rows and covers every required overload, null/empty input, numeric boundaries, wrong arity, unsupported types, aliases, distinct aggregates, lambda value/index binding, and attempts to use an unlisted name or bypass the manifest through `CallFunction`. Finalization is blocked until these rows and tests are published under the whole-bundle digest.
