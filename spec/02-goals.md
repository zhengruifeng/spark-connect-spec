# 2. Goals

## 2.1 History

Spark Connect was introduced in Apache Spark 3.4 (April 2023\) as an experimental client-server interface, motivated by the need to decouple Spark application code from Spark engine deployment. The protocol was developed in the open via the SPARK JIRA and committed to `apache/spark` directly, without a separate specification document.\
Through Apache Spark 3.5, 4.0, and 4.1, the Spark Connect API surface expanded substantially: Streaming, Machine Learning, User-Defined Functions, Artifacts, Reattachable Execution, and Catalog operations were added. Throughout this period the proto IDL — together with its inline comments and the behavior of the Apache Spark reference implementation — served as the de facto contract.\
By 2026, multiple vendor and partner systems had been built on top of Spark Connect. Behavioral drift between implementations had begun to produce incidents (notably in Delta DML semantics across server implementations) that demonstrated the cost of an implicit contract.\
This specification formalizes the Spark Connect contract. Version 1.0 reflects the state of the protocol as of Apache Spark 4.2.0.

## 2.2 Overview of Goals

These goals explain the design. They do not create requirements; §6 and the canonical `SC-1.0-P1` rows do.

1. **State testable requirements.** Normative rules identify observable behavior (§6.2).\
2. **Keep one conformance bar.** Conformance covers the complete Required Profile, not independently claimable modules (§6.3).\
3. **Standardize the protocol, not an engine's complete SQL dialect.** The core defines portable RPC, plan, result, session, error, type, relation, expression, and small query-text behavior (Appendix I).\
4. **Cover portable batch plans broadly.** Chapters 16 and 17 define the required relation and expression surfaces and their exclusions.\
5. **Keep name-based and textual semantics closed.** Appendices C and I enumerate the function and grammar rows; source registries do not expand them.\
6. **Permit independent full SQL dialects.** Complete engine dialects are optional profiles outside the Portable SQL Core (Appendix I.7).\
7. **Specify behavior, not only bytes.** The semantic chapters define portable observable results for required wire rows.\
8. **Keep the TCK protocol-first.** Core relation and expression tests construct protobuf plans directly; only Portable SQL and `ExpressionString` cases exercise required text (§6.3 and Appendix J.5).\
9. **Evolve wire surfaces additively.** Chapter 22 defines compatible additions and deprecation.\
10. **Freeze every released profile.** A release binds an immutable reference commit, canonical bundle, digest, and TCK revision (§22.1).\
11. **Use explicit extension points.** Wire extensions use the defined `google.protobuf.Any` fields and distinct type URLs (§22.3).\
12. **Bound reference fallback.** Apache Spark behavior applies only where a canonical row or normative section expressly delegates to it (§6.1.1).
