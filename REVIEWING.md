# Reviewing the Spark Connect Specification

This guide is informative. It organizes review work but adds no conformance requirements. The milestones are adapted from the [v0.49 Short Review Draft](https://docs.google.com/document/d/16glco6U_6kkfjsG7rnbtHpZDUefN6KdSclN-UGV0z2Y/edit).

> **Critical path:** The canonical `sc-1.0-p1.json` bundle has not been generated or published, and its publication commit and SHA-256 are still unset. Reviewers can assess the proposed membership, semantics, and TCK design in this draft, but cannot verify any required row or run a self-contained conformance review until that bundle lands. Bundle generation and publication therefore control the path to Final status.

**Start here:** [Review milestones](#review-milestones).\
**Suggested review order: begin with Milestone 1, then review Milestones 2–6 in order. Appendix K's contributor SOP is reviewed with the TCK, publication, and evolution rules in Milestone 6.**

## Changes in draft v0.49

* Retired the standalone Milestone 1 packet after confirming that it had no review comments and its structure-only scope duplicated the maintained drafts.\
* Consolidated its useful structure, navigation, figure, and milestone-ownership checks into Milestone 1 of the short review draft and removed packet links.\
* Kept Milestone 1 structure-only and Appendix K's informative contributor SOP in Milestone 6. Updated the two maintained drafts together and made no normative conformance change.

## Scope

Version 1.0 covers the core protocol, logical sessions, plan analysis and execution, errors, Arrow results, required data types, portable batch relations and expressions, configuration, Catalog basics, providers, client APIs, and the query-only Portable SQL Core. Authentication, authorization, credential formats, principal derivation, identity propagation, permission policy, and security certification are outside conformance. §6.3 is the sole conformance-membership summary.

## Deferred from v1.0

* UDF execution, artifacts, Structured Streaming, Spark Declarative Pipelines, and ML/MLlib algorithm contracts are deferred to later specification versions.\
* This guide does not enumerate every Spark API absent from the v1.0 Required Profile. Section 6 and the canonical bundle define membership.

## Open specification work

* **TODO-SPARK-SQL-SPEC:** Publish a separately versioned complete Spark SQL dialect specification and TCK. The query-only Portable SQL Core supports core SQL-related tests but does not replace a complete Spark SQL specification. Under the current proposal, this separate TODO does not block core v1.0 finalization; §22.1.3 remains authoritative.

## Normative sources and reading order

1. Read §6 for conformance policy, authority, and the complete Required Profile identifier `SC-1.0-P1`.\
2. Use the proto IDL at the pinned reference commit for message names, field numbers and types, enums, and RPC signatures.\
3. Use Chapters 7–21 for observable semantics of rows assigned to each chapter.\
4. Use Appendices C, H, and I for the closed function, provider, Portable SQL, and `ExpressionString` surfaces. Appendix I now reproduces every required grammar production.\
5. Use Appendix J for canonical-bundle serialization and TCK binding. Appendices and the companion Google Sheet are review views; they cannot add profile membership.

If two controlling sources disagree, §6.1.1 treats that disagreement as a release blocker. Source discovery and undocumented behavior do not add requirements.

## Finalization blockers

Section 22.1.3 is the sole complete finalization checklist. Reviewers should use that section for current publication blockers; this guide adds no separate conditions.

## Design decisions requiring ratification

Before Final status, reviewers must explicitly ratify two v1.0 choices:

* Appendix I defines a new query-only Portable SQL Core for parameterized cross-engine Spark Connect clients. It does not claim SQL-92 Entry Level. Reviewers should confirm that the named workload justifies both the included pipeline and the deferred syntax.\
* `FetchErrorDetails` on an unknown non-tombstoned session creates a fresh session before returning empty details. The IDL does not require creation; §7.4.1 retains it to match Apache Spark 4.2.0. Reviewers should ratify that compatibility choice or require a coordinated profile and TCK change.

Sections 16.5–16.6 now state the required aggregate and ranking-join results directly. Finalization still requires the canonical rows and TCK cases to encode those rules without adding semantics.

## Review milestones

Review one milestone at a time. Complete Milestone 1 before beginning technical review. Label comments and issues `M1` through `M6`.

For each milestone, record one outcome:

* **Accepted**
* **Accepted with named edits**
* **Blocked**

A later review may reopen an earlier decision when it finds a concrete conflict.

### Milestone 1: Document structure

**Purpose:** Approve the document organization and the sequence of later reviews.

**Read:**

* [Preface and original contents](spec/preface.md)
* [Chapter 1: Introduction](spec/01-introduction.md)
* [Chapter 2: Goals](spec/02-goals.md)
* [Chapter 3: Summary of First Release](spec/03-summary-of-first-release.md)
* [Chapter 4: Overview](spec/04-overview.md)
* Check only the placement and links of [Appendix E](spec/appendices/e-glossary.md) and [Appendix F](spec/appendices/f-related-documents.md).

**Focus:** Confirm that each opening section has one clear role, the specification and reference implementation are distinguished, every chapter and appendix has one milestone owner, and navigation and figures support bounded review. This milestone does not approve technical semantics or the contributor procedure.

**Output:** An accepted document map and a named list of structural, ownership, or navigation edits.

**Exit:** Reviewers can identify one role and one milestone owner for every chapter and appendix without gaps or overlaps.

### Milestone 2: Scope and authority

**Purpose:** Agree on what a Spark Connect Specification 1.0 conformance claim means.

**Read:**

* [Chapter 5: Services and Messages](spec/05-services-and-messages.md)
* [Chapter 6: Compliance](spec/06-compliance.md)
* [Chapter 20: Streaming](spec/20-streaming.md) and [Chapter 21: Machine Learning](spec/21-machine-learning.md) only as deferred-area boundaries
* Cross-read §22.1 in [Chapter 22](spec/22-versioning-and-evolution.md) only for version separation.

**Focus:** Check the bounded first-release scope, complete-profile rule, governing source for each kind of question, and separation between Specification 1.0 and Apache Spark 4.2.0.

**Output:** An accepted scope-and-authority statement and any explicitly deferred follow-ups.

**Exit:** Independent reviewers applying the precedence rules reach the same governing source and membership answer.

### Milestone 3: Protocol lifecycle

**Purpose:** Confirm that independent clients and servers can complete and diagnose the same RPC lifecycle.

**Read:**

* [Chapter 7: Errors](spec/07-errors.md)
* [Chapter 8: Sessions](spec/08-sessions.md)
* [Chapter 9: Plan Analysis](spec/09-plan-analysis.md)
* [Chapter 10: Execution](spec/10-execution.md)
* [Chapter 11: Configuration](spec/11-configuration.md)
* [Chapter 12: Catalog](spec/12-catalog.md)
* [Chapter 13: Artifacts](spec/13-artifacts.md)
* [Chapter 14: Result Sets and Arrow Encoding](spec/14-result-sets-and-arrow-encoding.md)
* [Appendix D: SQLState Code Reference](spec/appendices/d-sqlstate-code-reference.md)

**Focus:** Review identifiers, sessions, analysis, execution, streamed results, errors, interruption, release, reattachment, Arrow results, configuration, and Catalog behavior.

**Output:** An agreed success-path lifecycle and a failure-path checklist.

**Exit:** A TCK author can derive observable assertions for each required lifecycle transition without undocumented implementation details.

### Milestone 4: Portable plan semantics

**Purpose:** Review behavior that determines portable schemas, results, and failures.

**Read:**

* [Chapter 15: Data Types](spec/15-data-types.md)
* [Chapter 16: Relations](spec/16-relations.md)
* [Chapter 17: Expressions](spec/17-expressions.md)
* [Chapter 18: User-Defined Functions](spec/18-user-defined-functions.md)
* [Chapter 19: Commands](spec/19-commands.md)
* [Appendix B: Data Type Conversion Tables](spec/appendices/b-data-type-conversion-tables.md)
* [Appendix C: Built-in Function Reference](spec/appendices/c-built-in-function-reference.md)
* [Appendix G: Client API Membership Manifest](spec/appendices/g-client-api-membership-manifest.md)
* [Appendix H: Data-Source Provider Profile](spec/appendices/h-data-source-provider-profile.md)

**Focus:** Check types, relations, expressions, commands, providers, client operations, boundary cases, malformed shapes, and explicit source-validation obligations.

**Output:** A semantics issue list keyed to proposed Required Profile rows.

**Exit:** Every proposed required row has an identifiable semantic rule and test obligation, without silent reliance on unspecified Spark behavior.

### Milestone 5: SQL test surface

**Purpose:** Review Portable SQL as the small dialect-neutral input language for core SQL-related TCK cases.

**Read:**

* [Appendix I: Portable SQL and Expression-String Syntax](spec/appendices/i-portable-sql-and-expression-string-syntax.md)
* Cross-read §6.4 in [Chapter 6](spec/06-compliance.md), §10.5 in [Chapter 10](spec/10-execution.md), §17.6 in [Chapter 17](spec/17-expressions.md), and §19.3 in [Chapter 19](spec/19-commands.md).

**Focus:** Check whether the syntax and function floor is sufficient for common tests, remains independent of a backend's complete SQL dialect, and keeps the future complete Spark SQL specification as separate work.

**Output:** An accepted Portable SQL surface and a list of constructs to add, remove, or clarify before freezing the TCK.

**Exit:** Core SQL-related TCK inputs need no backend-specific rewrite, while additional backend syntax does not affect conformance.

### Milestone 6: TCK, publication, evolution, and contributor workflow

**Purpose:** Confirm that results are reproducible, tied to immutable artifacts, and maintained through a complete change procedure.

**Read:**

* [Chapter 22: Versioning and Evolution](spec/22-versioning-and-evolution.md)
* [Appendix A: Revision History](spec/appendices/a-revision-history.md)
* [Appendix J: Canonical Conformance Bundle](spec/appendices/j-canonical-conformance-bundle.md)
* [Appendix K: Contributor Change Procedure](spec/appendices/k-contributor-change-procedure.md)
* [TCK repository](https://github.com/zhengruifeng/spark-connect-tck)

**Focus:** Review bundle identity, hashes, manifest and report binding, adapter behavior, evidence, finalization gates, compatibility checks, coordinated updates, and immutable publication records. Keep Appendix K's informative workflow distinct from Chapter 22's normative gates.

**Output:** An accepted contributor workflow and a release-readiness checklist in which every unmet gate names an artifact, test, owner, or follow-up issue.

**Exit:** An independent reviewer can reproduce the artifact trace and TCK report, and no finalization blocker remains unresolved.
