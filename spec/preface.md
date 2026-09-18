# Spark Connect Specification

**Version:** 1.0\
**Status: Working Draft (v0.49)**\
**Specification Lead:** *TBD — named Apache Spark PMC sponsor required for SPIP*\
**Target repository:** `apache/spark`\
**SPIP:** *TBD*\
**Reference Spark version: Apache Spark 4.2.0 (v4.2.0, commit 32f7299601108917fb01920a54e084595b7b3bf8)**\
**Last updated: 2026-09-04**\
> This is a working draft for Spark Connect Specification 1.0. Do not cite it as an accepted Apache Spark specification until the SPIP and PMC approval recorded below are complete.

## Submit comments about this document at

`dev@spark.apache.org` and the `SPARK` JIRA project.\
---

## Contents

* **Preface**\
  * Typographic Conventions\
  * Submitting Feedback\
* **1. Introduction**\
* **2. Goals**\
* **3. Summary of First Release**\
* **4. Overview**\
* **5. Services and Messages**\
* **6. Compliance**\
* **7. Errors**\
* **8. Sessions**\
* **9. Plan Analysis**\
* **10. Execution**\
* **11. Configuration**\
* **12. Catalog**\
* **13. Artifacts**\
* **14. Result Sets and Arrow Encoding**\
* **15. Data Types**\
* **16. Relations**\
* **17. Expressions**\
* **18. User-Defined Functions**\
* **19. Commands**\
* **20. Streaming**\
* **21. Machine Learning**\
* **22. Versioning and Evolution**\
* **A. Revision History**\
* **B. Data Type Conversion Tables**\
* **C. Built-in Function Reference**\
* **D. SQLState Code Reference**\
* **E. Glossary**\
* **F. Related Documents**\
* **G. Client API Membership Manifest**\
* **H. Data-Source Provider Profile**\
* **I. Portable SQL and Expression-String Syntax**\
* **J. Canonical Conformance Bundle**\
* **K. Contributor Change Procedure (Informative SOP)**

## Preface

This document specifies Spark Connect, the client-server protocol of Apache Spark. It supersedes and consolidates:

* The proto-file source comments in `sql/connect/common/src/main/protobuf/`, which have until now served as the de facto contract.\
* Behavioral conventions established by the Apache Spark reference implementation and inherited by downstream distributions.\
* The undocumented expectations of partner systems (third-party Spark Connect Gateway implementations, vendor runtimes) operating against Spark Connect endpoints.

This document describes the protocol at a conceptual and normative level. Wire-format details and per-field reference are captured in the proto IDL shipped with the Apache Spark distribution. More extensive examples and reference tables are placed in the appendices.\
Readers can also reference the proto IDL for a complete and precise definition of each message and service:\
> https://github.com/apache/spark/tree/v4.2.0/sql/connect/common/src/main/protobuf

### Typographic Conventions

* `Monospace` denotes proto identifiers, RPC names, and code (e.g., `ExecutePlan`, `Relation.rel_type`).\
* *Italic* denotes new terms and definitions (e.g., a *session* is a logical execution context).\
* **Bold** denotes defined emphasis or labels (e.g., **Required interface**).\
* MUST, SHOULD, MAY and the other capitalized keywords denote normative language per RFC 2119 (e.g., "The server MUST return `OK`").

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174), when, and only when, they appear in all capitals.

### Submitting Feedback

Please send comments and questions concerning this specification to the Apache Spark developer mailing list at `dev@spark.apache.org`. File defects and editorial corrections as JIRA issues under the `SPARK` project with component `Connect`.\
---
