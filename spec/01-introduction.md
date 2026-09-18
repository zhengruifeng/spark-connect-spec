# 1. Introduction

## 1.1 The Spark Connect API

The Spark Connect API provides programmatic access to Apache Spark from a network-attached client. Using the Spark Connect API, applications can construct logical query plans, execute them against a remote Spark engine, and consume their results, without sharing a JVM or process with the engine.\
Spark Connect is based on Google Protocol Buffers and gRPC. It provides a vendor-neutral, language-neutral mapping from the Apache Spark SQL and DataFrame APIs to a wire protocol that can be implemented by clients in any language with a gRPC binding.\
Since its introduction in Apache Spark 3.4, the Spark Connect protocol has become the foundation for serverless Spark deployments, multi-tenant Spark services, and embedded Spark workloads.

## 1.2 Platforms

The Spark Connect API is a constituent technology of the Apache Spark platform. The protocol is implemented by:

* **Apache Spark**, as the reference server implementation, with first-party clients in PySpark (Python), the Spark Connect JVM Client (Scala/Java), and the Spark Connect JDBC Client.\
* **Vendor distributions** of Apache Spark that expose Spark Connect endpoints.\
* **Third-party clients** in languages such as Go and Rust that depend on the same proto IDL.

This specification is independent of any specific Spark version's feature set beyond what this document requires. Section 6.1.1 defines the limited cases in which the Apache Spark reference implementation controls behavior.

## 1.3 Target Audience

This specification is targeted primarily at:

* Implementers of Spark Connect **servers**, including Apache Spark itself, vendor distributions, and alternative engines exposing Spark Connect endpoints.\
* Implementers of Spark Connect **clients** in any language.\
* Authors of **tools and gateways** built on Spark Connect, including reverse proxies, query routers, observability layers, and migration utilities.

It is secondarily intended to serve:

* **Application developers** who need a precise reference for the runtime semantics of Spark Connect-based libraries.\
* **Authors of higher-level APIs** layered on top of Spark Connect.

## 1.4 Acknowledgements

*TBD: contributors, expert reviewers, and the named PMC sponsor.*\
---
