# I. Portable SQL and Expression-String Syntax

## I.1 Scope and provenance

`SC-1.0-P1-PORTABLE-SQL` defines the complete SQL-statement surface required by core Spark Connect 1.0. Its canonical `SQL-*` rows control lexical forms, query productions, parameter binding, function references, semantics, rejection behavior, and TCK coverage. The profile contains queries only. It does not contain DDL, DML, procedural statements, scripts, or administrative commands.

### I.1.1 Design target

The target consumer is a Spark Connect client or gateway that must send one parameterized, read-only query before it knows whether the endpoint is backed by Spark SQL or another engine. The client authors that query to this canonical grammar. Acceptance by every comparison engine does not add a Portable SQL form. For example, `ORDER BY x` is outside the profile because it omits explicit `NULLS FIRST` or `NULLS LAST`; an alias, when present, uses `AS`.\
The v1.0 workload is schema-oriented data access: literals and projection, filtering, inner, left, and cross joins, grouping with common aggregates, ordering, and bounded results. This floor supports portable client tests and simple application queries. It is not a general SQL application portability layer or a replacement for an engine dialect.

### I.1.2 Standards status

Portable SQL Core is a new Spark Connect subset. It does not claim SQL-92 Entry Level or any ISO SQL conformance level. JDBC could require SQL-92 Entry because it standardized access to SQL database systems. Spark Connect must also cover non-SQL logical plans, typed protobuf parameters, Arrow results, and backends whose accepted statement dialect is not SQL-92. Applying the SQL-92 Entry label would both import features absent here and imply compatibility that the TCK does not test.\
Apache Spark SQL 4.2.0, PostgreSQL 18, DuckDB, Trino, and JDBC's SQL-92 Entry requirement are comparison provenance only. They do not extend the grammar or settle a semantic disagreement. The grammar and rows in this appendix are the complete definition.

### I.1.3 Why this line

A construct enters v1.0 only when the target workload needs it, its behavior can be fixed across the comparison engines without dialect negotiation, it fits the closed type/function/collation profiles, and the TCK can cover both acceptance and rejection. The included query pipeline is sufficient to select, filter, combine, summarize, order, and page tabular data with typed parameters.\
The first release omits three categories deliberately:

* query structuring and advanced analytics (`WITH`, subqueries, windows, grouping syntax in SQL text), which are available through protobuf plans and need a larger SQL profile;\
* spellings with material cross-engine lexical or semantic differences (quoted identifiers, `LIKE` and escape rules, collations, engine type names, provider syntax); and\
* conveniences not needed by the baseline workload (`IN`, `BETWEEN`, right joins, and other forms that can be expressed with the included comparisons or by swapping join inputs).

Full outer joins and constructs that cannot be reduced without changing NULL or multiplicity semantics remain dialect-profile work. A later minor version may add a reviewed construct, but a released 1.0 profile never inherits syntax automatically.\
`SC-1.0-P1-EXPRESSION-SYNTAX` remains a separate, smaller grammar for `ExpressionString`. Canonical `EXPR-*` rows control that surface. Neither grammar imports an engine's full statement parser, keyword set, type system, or function registry.

## I.2 Portable SQL lexical elements

The grammar uses these conventions:

* text in double quotes is one exact token;\
* `"A" ... "Z"` denotes every code point in the inclusive range from `A` through `Z`;\
* keywords are case-insensitive;\
* braces mean zero or more repetitions, brackets mean optional content, and a vertical bar separates alternatives;\
* ASCII whitespace may occur between tokens but not inside a token; and\
* the lexer takes the longest token that matches at the current position.

The following lexical productions are normative. `unicode_string_character` means any Unicode scalar value except U+0000 and U+0027 APOSTROPHE. A doubled apostrophe inside a string denotes one apostrophe.\
ascii\_whitespace       ::= U+0009 | U+000A | U+000D | U+0020\
digit                  ::= "0" | "1" | "2" | "3" | "4"\
                           | "5" | "6" | "7" | "8" | "9"\
nonzero\_digit          ::= "1" | "2" | "3" | "4" | "5"\
                           | "6" | "7" | "8" | "9"\
ascii\_letter           ::= "A" ... "Z" | "a" ... "z"\
identifier\_start       ::= ascii\_letter | "\_"\
identifier\_continue    ::= identifier\_start | digit\
identifier\_part        ::= identifier\_start { identifier\_continue }\
identifier             ::= identifier\_part\
multipart\_identifier   ::= identifier\_part { "." identifier\_part }\
digits                 ::= digit { digit }\
positive\_integer       ::= nonzero\_digit { digit }\
nonnegative\_integer    ::= digits\
integer\_literal        ::= digits\
exponent\_part          ::= ("e" | "E") \["+" | "-"\] digits\
decimal\_literal        ::= digits "." digits \[ exponent\_part \]\
                           | digits exponent\_part\
numeric\_literal        ::= decimal\_literal | integer\_literal\
string\_unit            ::= unicode\_string\_character | "''"\
string\_literal         ::= "'" { string\_unit } "'"\
boolean\_literal        ::= "TRUE" | "FALSE"\
null\_literal           ::= "NULL"\
literal                ::= null\_literal | boolean\_literal\
                           | numeric\_literal | string\_literal\
named\_parameter        ::= ":" identifier\_part\
positional\_parameter   ::= "?"\
parameter\_marker       ::= named\_parameter | positional\_parameter\
Unquoted `identifier_part` tokens must not case-insensitively equal a reserved word. The reserved words are:\
`SELECT`, `DISTINCT`, `AS`, `VALUES`, `FROM`, `WHERE`, `GROUP`, `BY`, `HAVING`, `ORDER`, `LIMIT`, `OFFSET`, `UNION`, `ALL`, `INNER`, `JOIN`, `LEFT`, `OUTER`, `CROSS`, `ON`, `ASC`, `DESC`, `NULLS`, `FIRST`, `LAST`, `NULL`, `TRUE`, `FALSE`, `AND`, `OR`, `NOT`, `IS`, `CASE`, `WHEN`, `THEN`, `ELSE`, `END`, `CAST`, `BOOLEAN`, `SMALLINT`, `INTEGER`, `BIGINT`, `REAL`, `DOUBLE`, `DECIMAL`, `VARCHAR`, `DATE`, and `TIMESTAMP`.\
Portable SQL does not accept quoted identifiers, comments, statement terminators, hexadecimal or special floating-point literals, alternate string escapes, leading-dot or trailing-dot decimals, multiple statements, or alternate parameter markers. A sign is a unary operator, not part of a numeric token. Unquoted identifier resolution follows the required session case-sensitivity row.\
Function spellings listed in the `scalar_function_call`, `aggregate_function_call`, `expr_scalar_function_call`, `expr_aggregate_function_call`, and `expr_transform_call` productions are contextual function names, not reserved words. In those productions, a quoted alphabetic function terminal matches one `identifier_part` token case-insensitively when the next non-whitespace token is `(`. Outside a matching function-call production, the same spelling remains an `identifier_part` unless I.2 lists it as reserved. This contextual match does not reserve the spelling in table, column, attribute, alias, or lambda-variable positions.

## I.3 Query grammar

The following EBNF is normative. Every lexical reference is defined in I.2; `expression` is defined in I.4.\
query\_statement       ::= query\_body \[ order\_by\_clause \]\
                          \[ limit\_clause \] \[ offset\_clause \]\
query\_body            ::= query\_term { "UNION" "ALL" query\_term }\
query\_term            ::= select\_query | values\_query\
                          | "(" query\_statement ")"

select\_query          ::= "SELECT" \[ "DISTINCT" \] select\_item\
                          { "," select\_item }\
                          \[ from\_clause \] \[ where\_clause \]\
                          \[ group\_by\_clause \] \[ having\_clause \]\
select\_item           ::= "\*" | expression \[ "AS" identifier \]

values\_query          ::= "VALUES" values\_row { "," values\_row }\
values\_row            ::= "(" expression { "," expression } ")"

from\_clause           ::= "FROM" joined\_table\
joined\_table          ::= table\_primary { join\_clause }\
table\_primary         ::= multipart\_identifier \[ table\_alias \]\
                          | "(" query\_statement ")" table\_alias\
table\_alias           ::= "AS" identifier\
                          \[ "(" identifier { "," identifier } ")" \]\
join\_clause           ::= \[ "INNER" \] "JOIN" table\_primary\
                          "ON" boolean\_expression\
                          | "LEFT" \[ "OUTER" \] "JOIN" table\_primary\
                          "ON" boolean\_expression\
                          | "CROSS" "JOIN" table\_primary

where\_clause          ::= "WHERE" boolean\_expression\
group\_by\_clause       ::= "GROUP" "BY" expression { "," expression }\
having\_clause         ::= "HAVING" boolean\_expression\
order\_by\_clause       ::= "ORDER" "BY" sort\_item { "," sort\_item }\
sort\_item             ::= expression \[ "ASC" | "DESC" \]\
                          "NULLS" ( "FIRST" | "LAST" )\
limit\_clause          ::= "LIMIT" nonnegative\_integer\
offset\_clause         ::= "OFFSET" nonnegative\_integer\
boolean\_expression    ::= expression\
`UNION ALL` is the only required set operator. Derived tables require `AS` and an alias. A derived-table column list, when present, must have the relation's exact arity. Comma joins, `NATURAL JOIN`, `USING`, lateral syntax, and right or full joins are not Portable SQL forms; equivalent behavior remains available through required protobuf relations.

## I.4 Portable SQL expressions and functions

Portable SQL expressions use this complete normative grammar. It references only lexical productions from I.2.\
expression                    ::= or\_expression\
or\_expression                 ::= and\_expression { "OR" and\_expression }\
and\_expression                ::= not\_expression { "AND" not\_expression }\
not\_expression                ::= "NOT" not\_expression\
                                  | predicate\_expression\
predicate\_expression          ::= additive\_expression\
                                  \[ comparison\_operator additive\_expression\
                                  | "IS" \[ "NOT" \] "NULL" \]\
comparison\_operator           ::= "=" | "\<\>" | "\!=" | "\<" | "\<="\
                                  | "\>" | "\>="\
additive\_expression           ::= multiplicative\_expression\
                                  { ( "+" | "-" ) multiplicative\_expression }\
multiplicative\_expression     ::= unary\_arithmetic\_expression\
                                  { ( "\*" | "/" | "%" )\
                                  unary\_arithmetic\_expression }\
unary\_arithmetic\_expression   ::= ( "+" | "-" )\
                                  unary\_arithmetic\_expression\
                                  | primary\_expression\
primary\_expression            ::= literal | parameter\_marker\
                                  | multipart\_identifier\
                                  | required\_function\_call\
                                  | case\_expression | cast\_expression\
                                  | "(" expression ")"

required\_function\_call        ::= scalar\_function\_call\
                                  | aggregate\_function\_call\
scalar\_function\_call          ::= "ABS" "(" expression ")"\
                                  | "COALESCE" "(" expression "," expression\
                                    { "," expression } ")"\
                                  | "NULLIF" "(" expression "," expression ")"\
                                  | ( "LOWER" | "UPPER" | "LENGTH" | "TRIM" )\
                                    "(" expression ")"\
                                  | "SUBSTRING" "(" expression "," expression\
                                    \[ "," expression \] ")"\
                                  | "CONCAT" "(" expression "," expression\
                                    { "," expression } ")"\
aggregate\_function\_call       ::= "COUNT" "(" "\*" ")"\
                                  | "COUNT" "(" expression ")"\
                                  | ( "SUM" | "AVG" | "MIN" | "MAX" )\
                                    "(" expression ")"

case\_expression               ::= "CASE" "WHEN" expression "THEN" expression\
                                  { "WHEN" expression "THEN" expression }\
                                  \[ "ELSE" expression \] "END"\
                                  | "CASE" expression "WHEN" expression\
                                  "THEN" expression\
                                  { "WHEN" expression "THEN" expression }\
                                  \[ "ELSE" expression \] "END"\
cast\_expression               ::= "CAST" "(" expression "AS" portable\_type ")"\
portable\_type                 ::= "BOOLEAN" | "SMALLINT" | "INTEGER"\
                                  | "BIGINT" | "REAL" | "DOUBLE"\
                                  | "DECIMAL" "(" positive\_integer ","\
                                    nonnegative\_integer ")"\
                                  | "VARCHAR" | "DATE" | "TIMESTAMP"\
Binary arithmetic, `AND`, and `OR` are left-associative. Unary arithmetic and `NOT` are right-associative. Comparison and null-test operators bind more strongly than `NOT`, are non-associative, and cannot chain. Parentheses override precedence. An expression used as `boolean_expression`, a searched-`CASE` condition, or an operand of `NOT`, `AND`, or `OR` must have required `Boolean` type; NULL follows the three-valued rules in I.5.

| Row ID | Strength, strongest to weakest | Associativity and required forms |
| :---- | :---- | :---- |
| `SQL-PREC-PRIMARY` | Primary | Literals, parameters, column references, required calls, `CASE`, `CAST`, and parentheses. |
| `SQL-PREC-UNARY` | Unary arithmetic | Unary `+` and `-`; right-associative. |
| `SQL-PREC-MULTIPLICATIVE` | Multiplicative | `*`, `/`, and `%`; left-associative. |
| `SQL-PREC-ADDITIVE` | Additive | `+` and `-`; left-associative. |
| `SQL-PREC-PREDICATE` | Comparison or null test | One comparison or `IS [NOT] NULL`; non-associative. |
| `SQL-PREC-NOT` | Boolean negation | `NOT`; right-associative and weaker than predicates. |
| `SQL-PREC-AND` | Boolean conjunction | `AND`; left-associative. |
| `SQL-PREC-OR` | Boolean disjunction | `OR`; left-associative. |

Portable type names map before backend parsing or translation:

| Portable SQL spelling | Required logical type | Required cast-row ID | Rule |
| :---- | :---- | :---- | :---- |
| `BOOLEAN` | `Boolean` | `CAST-SQL-BOOLEAN` | Boolean plus NULL. |
| `SMALLINT` | `Short` | `CAST-SQL-SMALLINT` | Signed 16-bit integer. |
| `INTEGER` | `Integer` | `CAST-SQL-INTEGER` | Signed 32-bit integer. |
| `BIGINT` | `Long` | `CAST-SQL-BIGINT` | Signed 64-bit integer. |
| `REAL` | `Float` | `CAST-SQL-REAL` | IEEE 754 binary32, not `Double`. |
| `DOUBLE` | `Double` | `CAST-SQL-DOUBLE` | IEEE 754 binary64. |
| `DECIMAL(p,s)` | `Decimal(p,s)` | `CAST-SQL-DECIMAL` | `1 <= p <= 38` and `0 <= s <= p`. |
| `VARCHAR` | `String` with `UTF8_BINARY` | `CAST-SQL-VARCHAR` | Unbounded portable string, not `VarChar(length)`. |
| `DATE` | `Date` | `CAST-SQL-DATE` | Calendar date without time or zone. |
| `TIMESTAMP` | `Timestamp` | `CAST-SQL-TIMESTAMP` | Microsecond instant parsed in the session zone; never `TimestampNTZ`. |

`spark.sql.timestampType` does not change the Portable SQL meaning of `TIMESTAMP`. A backend that rejects bare `VARCHAR` translates it to the required `String` type. Each type row has a corresponding `SQL-TYPE-*` and `CAST-SQL-*` row. A cast is required only when its canonical cast row admits the source type, target type, and evaluation mode.\
The grammar fixes function spelling and arity. The applicable `FN-*` row fixes argument domains, null behavior, and result type. `substring` uses a one-based positive start and optional nonnegative length. `concat` and `coalesce` require at least two arguments. `count(expression)` ignores NULL. The statement grammar does not admit `DISTINCT` inside an aggregate, the `substr` alias, `transform`, or any unlisted function.

## I.5 Portable query semantics

* Queries use bag semantics unless `DISTINCT` is written. `UNION ALL` preserves duplicates and operand order is not a result-order guarantee.\
* `WHERE`, join `ON`, and `HAVING` retain rows only when their Boolean condition is TRUE. FALSE and UNKNOWN do not pass.\
* Arithmetic, comparison, three-valued Boolean logic, NULL propagation, aggregate NULL handling, casts, and required function behavior follow Chapters 15 and 17 and the cited canonical rows.\
* `GROUP BY` accepts expressions, not ordinals, grouping sets, or select-list aliases. Every nonaggregate selected expression MUST be a grouping expression. `HAVING` applies after grouping.\
* An inner join emits matching pairs. A left join additionally emits each unmatched left row once with NULLs for the right side. A cross join emits the Cartesian product.\
* `UNION ALL` operands MUST have equal arity and already compatible types under Chapter 15; otherwise an explicit `CAST` is required. Output names come from the first operand.\
* Result order is unspecified without `ORDER BY`. Every portable sort item explicitly selects `NULLS FIRST` or `NULLS LAST`; no engine-specific default NULL ordering is imported. `LIMIT` and `OFFSET` are nonnegative integer literals and apply after ordering when ordering is present.\
* `DISTINCT` and grouping treat NULL values as not distinct for duplicate/group formation. String equality, ordering, case conversion, and length use the required string/collation rows.\
* Select-list aliases define output names. `ORDER BY` may reference an unambiguous output alias but not an ordinal. The TCK uses qualification when an output alias and input column would otherwise conflict. Names of unaliased computed expressions are not portable and are not asserted by the TCK.

## I.6 Parameters, transport, and results

A portable request uses either `Relation.SQL.named_arguments` with `:name` markers or `Relation.SQL.pos_arguments` with `?` markers. The two fields MUST NOT coexist. Positional arguments bind left to right and their count MUST equal the number of markers. Every distinct named marker MUST have exactly one map entry; repeated occurrences use the same value; missing or unused entries fail before execution.\
Portable parameter values are typed `Expression` messages evaluated as scalar expressions without row input. Literals and foldable combinations of required operators, casts, and functions are required. Attributes, aggregates, windows, subqueries, UDFs, and other row-dependent expressions are invalid parameter values. The server MUST bind the expression structurally and MUST NOT perform string substitution, quoting, or reparsing of its rendered value.\
Deprecated `args` and `pos_args` remain wire-compatible for legacy literal bindings. A core conformance case uses at most one parameter family and uses the current `named_arguments` or `pos_arguments` fields. Behavior for a request that mixes deprecated and current families is outside the Portable SQL Core and MUST NOT be used to claim a pass.\
Executing a portable `Relation.SQL` yields the ordinary schema, Arrow batches, metrics, and completion defined by Chapters 10 and 14. Executing `SqlCommand` with a portable SQL input yields `SqlCommandResult.relation` for the same lazy query, as §19.3 requires. Since portable statements are queries only, the core TCK has no eager SQL-command result case.

## I.7 Exclusions and complete dialect profiles

The Portable SQL Core excludes:

* `WITH`, recursive queries, DDL, DML, MERGE text, transaction control, administrative statements, scripts, and multiple statements;\
* `UNION` without `ALL`, `INTERSECT`, `EXCEPT`, right/full/natural/lateral joins, comma joins, `USING`, pivots, unpivots, sampling, and table-valued functions;\
* subquery expressions, `EXISTS`, `IN`, `BETWEEN`, `LIKE`, regex predicates, window syntax, `QUALIFY`, grouping sets, rollups, cubes, and aggregate filters;\
* quoted identifiers, ordinals in grouping or ordering, collations, hints, time travel, provider syntax, and type names or parameters absent from I.4, including CHAR(n), VARCHAR(n), FLOAT, STRING, and TIMESTAMP\_NTZ;\
* array/map/struct constructors, interval, Variant, geospatial, UDT, and complex-type cast syntax; and\
* every unlisted statement, production, operator, function, or parameter form.

A core-only server rejects an excluded form with the Chapter 7 envelope. A server MAY accept it under a complete, versioned dialect profile or as documented deployment behavior, but that acceptance does not add a core requirement. `SPARK-SQL-4.2` may define the complete Spark SQL 4.2 grammar, functions, and query-versus-command behavior as an optional profile. The core protocol has no RPC for discovering or negotiating such a profile.

## I.8 `ExpressionString` grammar

`ExpressionString` has a separate complete grammar. It reuses the lexical `literal`, `identifier_part`, `positive_integer`, and `nonnegative_integer` productions from I.2. `non_backtick_scalar` means any Unicode scalar value except U+0000 and U+0060 GRAVE ACCENT; a doubled grave accent denotes one grave accent inside a quoted attribute part.\
expr\_expression               ::= expr\_or\_expression\
expr\_or\_expression            ::= expr\_and\_expression\
                                  { "OR" expr\_and\_expression }\
expr\_and\_expression           ::= expr\_not\_expression\
                                  { "AND" expr\_not\_expression }\
expr\_not\_expression           ::= "NOT" expr\_not\_expression\
                                  | expr\_predicate\_expression\
expr\_predicate\_expression     ::= expr\_additive\_expression\
                                  \[ expr\_comparison\_operator\
                                    expr\_additive\_expression\
                                  | "IS" \[ "NOT" \] "NULL" \]\
expr\_comparison\_operator      ::= "=" | "==" | "\<\>" | "\!=" | "\<"\
                                  | "\<=" | "\>" | "\>=" | "\<=\>"\
expr\_additive\_expression      ::= expr\_multiplicative\_expression\
                                  { ( "+" | "-" )\
                                    expr\_multiplicative\_expression }\
expr\_multiplicative\_expression ::= expr\_unary\_expression\
                                  { ( "\*" | "/" | "%" )\
                                    expr\_unary\_expression }\
expr\_unary\_expression         ::= ( "+" | "-" ) expr\_unary\_expression\
                                  | expr\_primary\_expression\
expr\_primary\_expression       ::= literal | expr\_attribute\_reference\
                                  | expr\_required\_function\_call\
                                  | expr\_case\_expression\
                                  | expr\_cast\_expression\
                                  | "(" expr\_expression ")"

expr\_attribute\_reference      ::= expr\_attribute\_part\
                                  { "." expr\_attribute\_part }\
expr\_attribute\_part           ::= identifier\_part | backtick\_identifier\
backtick\_identifier           ::= "\`" backtick\_unit { backtick\_unit } "\`"\
backtick\_unit                 ::= non\_backtick\_scalar | "\`\`"

expr\_required\_function\_call   ::= expr\_scalar\_function\_call\
                                  | expr\_aggregate\_function\_call\
                                  | expr\_transform\_call\
expr\_scalar\_function\_call     ::= "ABS" "(" expr\_expression ")"\
                                  | "COALESCE" "(" expr\_expression ","\
                                    expr\_expression\
                                    { "," expr\_expression } ")"\
                                  | "NULLIF" "(" expr\_expression ","\
                                    expr\_expression ")"\
                                  | ( "LOWER" | "UPPER" | "LENGTH" | "TRIM" )\
                                    "(" expr\_expression ")"\
                                  | ( "SUBSTRING" | "SUBSTR" ) "("\
                                    expr\_expression "," expr\_expression\
                                    \[ "," expr\_expression \] ")"\
                                  | "CONCAT" "(" expr\_expression ","\
                                    expr\_expression\
                                    { "," expr\_expression } ")"\
expr\_aggregate\_function\_call  ::= "COUNT" "(" "\*" ")"\
                                  | "COUNT" "(" expr\_expression ")"\
                                  | ( "SUM" | "AVG" | "MIN" | "MAX" )\
                                    "(" expr\_expression ")"\
expr\_transform\_call           ::= "TRANSFORM" "(" expr\_expression ","\
                                  lambda\_expression ")"\
lambda\_expression             ::= lambda\_identifier "-\>" expr\_expression\
                                  | "(" lambda\_identifier ","\
                                    lambda\_identifier ")" "-\>" expr\_expression\
lambda\_identifier             ::= identifier\_part

expr\_case\_expression          ::= "CASE" "WHEN" expr\_expression "THEN"\
                                  expr\_expression\
                                  { "WHEN" expr\_expression "THEN"\
                                    expr\_expression }\
                                  \[ "ELSE" expr\_expression \] "END"\
                                  | "CASE" expr\_expression "WHEN"\
                                    expr\_expression "THEN" expr\_expression\
                                  { "WHEN" expr\_expression "THEN"\
                                    expr\_expression }\
                                  \[ "ELSE" expr\_expression \] "END"\
expr\_cast\_expression          ::= "CAST" "(" expr\_expression "AS"\
                                  expr\_primitive\_type ")"\
expr\_primitive\_type           ::= "TINYINT" | "SMALLINT" | "INT"\
                                  | "INTEGER" | "BIGINT" | "FLOAT"\
                                  | "DOUBLE" | "DECIMAL" "("\
                                    positive\_integer ","\
                                    nonnegative\_integer ")"\
                                  | "BOOLEAN" | "STRING" | "BINARY"\
                                  | "DATE" | "TIMESTAMP" | "TIMESTAMP\_NTZ"\
Unquoted attribute and lambda parts follow I.2's identifier and reserved-word rules. For `ExpressionString`, `TINYINT`, `INT`, `FLOAT`, `STRING`, `BINARY`, and `TIMESTAMP_NTZ` are added to I.2's reserved-word set. Backticks quote one attribute part only; the grammar does not accept double-quoted attributes. Whitespace may appear between tokens but not inside `->`, a multi-character comparison operator, a named lexical token, or a backtick escape.

| Row family | Strength, strongest to weakest | Required forms |
| :---- | :---- | :---- |
| `EXPR-PREC-PRIMARY` | Primary | Literal, attribute, required call, `CASE`, `CAST`, and parentheses. |
| `EXPR-PREC-UNARY` | Unary arithmetic | Unary `+` and `-`; right-associative. |
| `EXPR-PREC-MULTIPLICATIVE` | Multiplicative | `*`, `/`, and `%`; left-associative. |
| `EXPR-PREC-ADDITIVE` | Additive | `+` and `-`; left-associative. |
| `EXPR-PREC-PREDICATE` | Comparison or null test | One listed comparison or `IS [NOT] NULL`; non-associative. |
| `EXPR-PREC-NOT` | Boolean negation | `NOT`; right-associative and weaker than predicates. |
| `EXPR-PREC-AND` | Boolean conjunction | `AND`; left-associative. |
| `EXPR-PREC-OR` | Boolean disjunction | `OR`; left-associative. |
| `EXPR-LAMBDA` | `transform` argument | One-variable or `(value,index)` lambda. |

The applicable `FN-*` and `CAST-*` rows fix argument domains, null behavior, result types, and cast behavior. For `ExpressionString`, `TIMESTAMP` maps to `Timestamp` and `TIMESTAMP_NTZ` maps to `TimestampNTZ`; `spark.sql.timestampType` changes neither mapping. The grammar excludes parameter markers, SQL statements, table references, subqueries, windows, aliases, sort clauses, complex constructors and types, aggregate `DISTINCT`, lambdas outside `transform`, and every unlisted function or operator.

## I.9 Errors and TCK

An out-of-profile Portable SQL or `ExpressionString` input fails before execution with the Chapter 7 envelope and the canonical parse, unresolved-function, unsupported-feature, invalid-parameter, or analysis condition. A server MUST NOT execute a valid prefix, silently delete an unsupported clause, substitute another function, or reinterpret a negative core case under a more permissive dialect.\
The core TCK includes positive and negative cases for every `SQL-*` and `EXPR-*` row. Portable SQL cases cover both `Relation.SQL` and the `SqlCommand.input` path; `SELECT`, `VALUES`, table access, each required join/clause, `UNION ALL`, explicit NULL ordering, functions, named and positional parameters, result schemas/rows, and all exclusion boundaries. For every `SQL-TYPE-*` row, the TCK executes the corresponding `CAST-SQL-*` row and asserts the exact logical result type. It specifically verifies `REAL` as `Float`, bare `VARCHAR` as `String` with `UTF8_BINARY`, and `TIMESTAMP` as `Timestamp` under UTC and a non-UTC session time zone while varying `spark.sql.timestampType` when the implementation exposes that non-core setting. Negative cases reject `CHAR(n)`, `VARCHAR(n)`, `FLOAT`, `STRING`, `TIMESTAMP_NTZ`, invalid `DECIMAL` parameters, and cast pairs absent from the canonical rows.\
Portable SQL precedence cases cite the applicable `SQL-PREC-*` rows. They verify that `NOT 1 = 1` parses as `NOT (1 = 1)`, `8 - 3 - 2` as `(8 - 3) - 2`, and `16 / 4 / 2` as `(16 / 4) / 2`; they also reject unparenthesized chained predicates. Expression-string cases cite the applicable EXPR-PREC-\* rows, run inside directly constructed protobuf plans, and verify the same NOT/predicate and arithmetic-associativity boundaries with expression-specific operators, along with every production, literal escape, identifier form, primitive cast, required function call, and lambda form.\
A complete-dialect TCK binds to a separate immutable dialect artifact. It may reuse core transport and result cases, but it MUST NOT relabel dialect-only syntax or command behavior as `SC-1.0-P1-PORTABLE-SQL` coverage.\
The TCK also fixes contextual function-name behavior. Portable SQL accepts a fixture in which `abs` is both a table and a column: `SELECT abs FROM abs` resolves both identifier uses, while `SELECT abs(abs) FROM abs` invokes the required `ABS` function on that column. `ExpressionString` likewise accepts `abs` as an attribute and `abs(abs)` as a function call whose argument is that attribute. Equivalent mixed-case spellings verify case-insensitive function matching without reserving the identifier.
