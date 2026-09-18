# D. SQLState Code Reference

The SparkThrowable.sql\_state field uses ANSI SQL SQLSTATE codes (5-character strings). For v1.0, error-class names, SQLSTATE values, and message-parameter schemas are frozen to common/utils/src/main/resources/error/error-conditions.json at Apache Spark tag v4.2.0 commit 32f7299601108917fb01920a54e084595b7b3bf8. Later registry additions are not inherited.\
Common SQLSTATE classes used by Spark Connect include:

* `22000`–`2299X` — Data exceptions. Specific codes include `22003` (numeric value out of range), `22004` (null value not allowed), `22005` (error in assignment), `22008` (datetime field overflow), `22012` (division by zero), `22023` (invalid parameter value), and `2200G` (most specific type mismatch).\
* `42000`–`4299X` — Syntax errors and access rule violations. Specific codes include `42601` (syntax error), `42703` (undefined column), `42883` (undefined function), and `42P01` (undefined table).
