# 3. Summary of First Release

## 3.1 Overview of changes

Version 1.0 formalizes the Spark Connect contract at the Apache Spark 4.2.0 reference snapshot. It adds no RPC or wire field. This section is a reading map, not a second conformance list.

| Review question | Controlling section |
| :---- | :---- |
| What is required or excluded? | §6.3 and the canonical `SC-1.0-P1` bundle |
| What observable behavior is required? | Chapters 7–21, as assigned by canonical rows |
| Which functions and text forms are portable? | Appendices C and I |
| Which providers and client APIs are required? | Canonical provider/client rows; Appendices H and G are review views |
| How are versions frozen and finalized? | §22.1 |
| How is the canonical artifact serialized and bound to the TCK? | Appendix J |

The Reviewer Guide summarizes scope and work deferred from v1.0. Complete Spark SQL, PostgreSQL-compatible SQL, and other engine dialects remain optional profiles; they do not expand the Portable SQL Core.
