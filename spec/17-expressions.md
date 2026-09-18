# 17. Expressions

An `Expression` represents a scalar, aggregate, window, lambda, subquery, or command-support value. Its `expr_type` oneof selects one variant. `ExpressionCommon.origin` is diagnostic and MUST NOT change evaluation.

## 17.1 Common expression contract

Every expression MUST select exactly one variant and provide its required children in semantic order. A server MUST reject an invalid enum, missing child, contradictory field, unsupported variant, or reference outside the current plan/session.\
Analysis determines data type, nullability, resolution, coercion, foldability, and determinism. Optimizers may rewrite an expression only when the rewrite preserves required values and errors.

## 17.2 Required v1.0 expression profile

The core profile requires every portable standard expression variant below:

* `Literal`, `UnresolvedAttribute`, `UnresolvedStar`, and `UnresolvedRegex`;\
* constrained `UnresolvedFunction` and constrained `ExpressionString`;\
* `Alias`, `Cast`, `SortOrder`, `Window`, and `NamedArgumentExpression`;\
* `LambdaFunction` and `UnresolvedNamedLambdaVariable`;\
* `UnresolvedExtractValue` and `UpdateFields`;\
* `SubqueryExpression`; and\
* `MergeAction` only inside required merge commands.

The following variants are outside the core profile:

* `CommonInlineUserDefinedFunction`, including Python, Scala, and Java UDF payloads;\
* `TypedAggregateExpression`;\
* `CallFunction`, because an unconstrained name would bypass `SC-1.0-P1-FUNCTIONS`;\
* internal `DirectShufflePartitionID`; and\
* `extension`.

An implementation may support an excluded expression, but it cannot use that expression to pass a core test.

## 17.3 Literals

`Literal.literal_type` selects the wire representation. A typed null and every complex literal include enough `DataType` information to determine the value unambiguously.\
Integer widths, floating values, decimal precision/scale, date epoch days, timestamp microseconds, binary bytes, and nested element order are normative. Array element types must agree. Map key and value counts must match; map keys are non-null. Struct values preserve field order.\
A literal using a type outside Chapter 15 is outside the core profile even when the literal variant can encode it.

## 17.4 Attributes, stars, regex, and extraction

`UnresolvedAttribute.unparsed_identifier` uses the `ExpressionString` identifier production in Appendix I.8. Optional `plan_id` narrows resolution to the corresponding plan. `is_metadata_column=true` requests metadata-column resolution and MUST NOT fall back to an ordinary column with the same spelling.\
An absent `UnresolvedStar.unparsed_target` expands the applicable scope. A present target is a qualified-star identifier. `UnresolvedRegex` applies the canonical regular-expression column selection rule and optional plan identity.\
`UnresolvedExtractValue` requires child and extraction expressions and supports required array, map, and struct domains. Missing or ambiguous resolution is a structured error; the server MUST NOT choose an arbitrary match.

## 17.5 `UnresolvedFunction` kernel

A core `UnresolvedFunction` uses an unqualified canonical name, `is_user_defined_function=false`, and absent or false `is_internal`. Resolution is case-insensitive. The invocation must match one canonical `FN-*` overload row.\
Required operator names are:

* arithmetic: `+`, `-`, `*`, `/`, `%`, and unary `negative`;\
* comparison: `==`, `!=`, `<`, `<=`, `>`, `>=`, and null-safe `<=>`;\
* Boolean: `and`, `or`, and `not`; and\
* null tests: `isNull` and `isNotNull`.

Required function names are:

* scalar: `abs`, `coalesce`, `nullif`, `lower`, `upper`, `length`, `substring`, `substr`, `concat`, and `trim`;\
* aggregate: `count`, `sum`, `avg`, `min`, and `max`; and\
* higher order: `transform`.

Appendix C fixes arity, type domains, return types, nullability, determinism, aliases, and errors. An unlisted name or overload is outside the profile. A server MUST NOT satisfy it by calling a UDF, applying an optional-type cast, or routing through `CallFunction`.\
`NamedArgumentExpression` is valid only when the selected `FN-*` row names that argument form. Duplicate names and invalid positional/named mixing are analysis errors.

## 17.6 `ExpressionString`

`ExpressionString.expression` uses only the closed grammar in Appendix I.8. Required forms are literals, identifiers, parentheses, arithmetic, comparisons, Boolean logic, null tests, `CASE`, primitive casts, calls to the §17.5 function set, and lambdas used by `transform`.\
The core grammar is independent of the server's statement-SQL dialect. A server MUST NOT accept an out-of-profile expression as a core conformance success merely because its SQL parser understands it. Client data is represented by literals or typed plan expressions, not concatenated into expression text.

## 17.7 Alias and struct updates

`Alias` requires a child and one or more name parts. A scalar alias normally has one part. Optional metadata is a valid JSON object.\
`UpdateFields` requires a struct child and field name. A present value adds or replaces the field; an absent value drops it. The canonical rows define nested-name, duplicate-field, and missing-field behavior.

## 17.8 Cast

`Cast` requires one child and exactly one target: structured `DataType` or `type_str`. Core `type_str` uses the `ExpressionString` primitive type names in Appendix I.8.\
`LEGACY`, `ANSI`, and `TRY` retain their distinct overflow and invalid-input behavior. `UNSPECIFIED` uses the required session/configuration row; it is not always legacy. The canonical `CAST-*` rows control source/target membership and boundary behavior.

## 17.9 Sort order

`SortOrder` requires a child, direction, and null ordering. It is an ordering descriptor and MUST NOT be accepted as an ordinary projected scalar. A server preserves each explicit ascending/descending and nulls-first/nulls-last combination.

## 17.10 Window expressions

`Window` requires a window function; partition and order specifications are optional. Core window functions are the required aggregate functions from §17.5. Ranking, analytic, and other unlisted functions require an optional function profile.\
An explicit frame requires `ROW` or `RANGE` plus lower and upper boundaries. Current-row, unbounded, and value boundaries preserve their distinct meaning. The server validates boundary order, value type, RANGE ordering, and correlation under the canonical rows.

## 17.11 Lambdas and `transform`

`LambdaFunction` requires a body and one or two variables for the core `transform` overloads. Variables use `UnresolvedNamedLambdaVariable` and lexical binding. A lambda variable MUST NOT capture an unrelated input column because the spelling matches.\
Core `transform` supports value-only and value-plus-index lambdas over required array element types. Python/Scala closures and worker-backed lambdas are excluded.

## 17.12 Subqueries

`SubqueryExpression` requires a current-plan `plan_id` and a non-unknown type: scalar, exists, table argument, or in-subquery. `IN` preserves comparison values. Scalar cardinality, correlation, null semantics, table-argument partition/order fields, and invalid outer references follow the canonical rows.\
A plan ID is not a cross-session handle.

## 17.13 Merge actions

`MergeAction` is valid only inside `MergeIntoTableCommand`. Delete carries an optional condition and no assignments. Insert and update carry assignments unless their star form applies. Every assignment requires key and value expressions.\
The server validates action type, conditions, duplicate targets, and resolution before applying the action.

## 17.14 Nulls, errors, and exclusions

Required expressions use three-valued Boolean logic, the canonical NaN rules, session ANSI behavior, and structured errors. An optimizer MUST NOT reorder evaluation when that changes a required error or a nondeterministic result.\
An unsupported or out-of-profile expression MUST fail explicitly unless the implementation documents it as an extension. A server MUST NOT replace it with null, its first child, a text rendering, `CallFunction`, or a UDF.

## 17.15 Expression TCK requirements

The core TCK constructs expression protobufs directly and covers:

* every required expression variant and every `FN-*` overload;\
* typed nulls, numeric boundaries, complex literals, and null/NaN behavior;\
* missing, ambiguous, qualified, metadata, star, regex, and extraction resolution;\
* every required arithmetic, comparison, Boolean, null-test, scalar, aggregate, and `transform` row;\
* wrong arity, unsupported type domains, unlisted names, and attempts to bypass the manifest with `CallFunction`;\
* all cast modes and every sort/null-order combination;\
* ROW and RANGE frames with required aggregate functions;\
* lambda shadowing and index variables;\
* scalar, exists, in, and table-argument subqueries;\
* the complete Appendix I.8 `EXPR-*` positive and negative grammar corpus; and\
* unsupported UDF, typed-aggregate, internal, and extension rejection.

Portable SQL is tested separately under Appendix I.9. It is not used to manufacture a pass for a required protobuf expression case.
