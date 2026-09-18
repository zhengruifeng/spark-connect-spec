# 18. User-Defined Functions

*Reserved for a future version of this specification.*\
This chapter will define the UDF surface: Python UDFs (scalar, Pandas variants, `mapInPandas` / `applyInPandas`, UDTF), Scala/Java UDFs, UDF dependency on Artifacts, and error propagation from worker → engine → client.\
UDFs are deferred from v1.0. The implementation pattern is well-understood — a non-Spark engine spawns Python worker processes and reuses the open-source PySpark worker protocol (`python/pyspark/worker.py`, `PythonRunner.scala`) — but UDF support pulls Artifacts (Chapter 13\) into scope, requires bundling a PySpark library so user UDF code can `import pyspark.sql.functions`, and carries a PySpark-version alignment burden. Pulling all of that into v1.0 enlarges the conformance scope materially, so it is deferred to a later spec version.\
A v1.0-conforming server MAY implement UDFs ahead of the spec. Such behavior is outside SC-1.0-P1; v1.0 fixes only the pinned IDL wire shape and makes no portable worker, artifact, numerical, error, or future-compatibility claim for the implementation. The associated proto messages (`CommonInlineUserDefinedFunction`, `CommonInlineUserDefinedTableFunction`, `CommonInlineUserDefinedDataSource`, and the related relation variants `MapPartitions`, `GroupMap`, `CoGroupMap`) are not part of the v1.0 conformance surface.\
---
