# Spark Connect Specification 1.0

> **Working Draft v0.49.** This proposal is hosted in a personal repository. It has not been accepted as an Apache Spark specification and must not be cited as one.

This repository is a chapter-by-chapter Markdown import of the Spark Connect Specification 1.0 working draft. The split makes reviews smaller, line-based, and easy to track through pull requests.

- **Reference implementation:** [Apache Spark 4.2.0 at commit `32f7299601108917fb01920a54e084595b7b3bf8`](https://github.com/apache/spark/commit/32f7299601108917fb01920a54e084595b7b3bf8)
- **Review plan:** [REVIEWING.md](REVIEWING.md)
- **TCK repository:** [spark-connect-tck](https://github.com/zhengruifeng/spark-connect-tck)
- **License:** [Apache License 2.0](LICENSE)

## Status and authority

The initial Markdown import preserves the v0.49 source draft and changes only its layout and Markdown formatting. An unintended difference from that import is a defect. Future Apache publication and authority remain subject to the SPIP and PMC process described by the draft.

The proposed conformance authorities are defined in [Chapter 6](spec/06-compliance.md), not by this README:

- The future canonical bundle identifies required-profile membership and row-level constraints.
- The protobuf IDL at the pinned Apache Spark commit controls wire shapes.
- The specification's normative prose defines portable observable semantics for selected members.
- The TCK records conformance evidence; it does not create requirements.

The canonical bundle, its publication commit, and its digest have not yet been published. The draft therefore cannot yet support a final conformance claim.

## Specification

- [Preface and original contents](spec/preface.md)
- [Chapter 1: Introduction](spec/01-introduction.md)
- [Chapter 2: Goals](spec/02-goals.md)
- [Chapter 3: Summary of First Release](spec/03-summary-of-first-release.md)
- [Chapter 4: Overview](spec/04-overview.md)
- [Chapter 5: Services and Messages](spec/05-services-and-messages.md)
- [Chapter 6: Compliance](spec/06-compliance.md)
- [Chapter 7: Errors](spec/07-errors.md)
- [Chapter 8: Sessions](spec/08-sessions.md)
- [Chapter 9: Plan Analysis](spec/09-plan-analysis.md)
- [Chapter 10: Execution](spec/10-execution.md)
- [Chapter 11: Configuration](spec/11-configuration.md)
- [Chapter 12: Catalog](spec/12-catalog.md)
- [Chapter 13: Artifacts](spec/13-artifacts.md)
- [Chapter 14: Result Sets and Arrow Encoding](spec/14-result-sets-and-arrow-encoding.md)
- [Chapter 15: Data Types](spec/15-data-types.md)
- [Chapter 16: Relations](spec/16-relations.md)
- [Chapter 17: Expressions](spec/17-expressions.md)
- [Chapter 18: User-Defined Functions](spec/18-user-defined-functions.md)
- [Chapter 19: Commands](spec/19-commands.md)
- [Chapter 20: Streaming](spec/20-streaming.md)
- [Chapter 21: Machine Learning](spec/21-machine-learning.md)
- [Chapter 22: Versioning and Evolution](spec/22-versioning-and-evolution.md)

## Appendices

- [Appendix A: Revision History](spec/appendices/a-revision-history.md)
- [Appendix B: Data Type Conversion Tables](spec/appendices/b-data-type-conversion-tables.md)
- [Appendix C: Built-in Function Reference](spec/appendices/c-built-in-function-reference.md)
- [Appendix D: SQLState Code Reference](spec/appendices/d-sqlstate-code-reference.md)
- [Appendix E: Glossary](spec/appendices/e-glossary.md)
- [Appendix F: Related Documents](spec/appendices/f-related-documents.md)
- [Appendix G: Client API Membership Manifest](spec/appendices/g-client-api-membership-manifest.md)
- [Appendix H: Data-Source Provider Profile](spec/appendices/h-data-source-provider-profile.md)
- [Appendix I: Portable SQL and Expression-String Syntax](spec/appendices/i-portable-sql-and-expression-string-syntax.md)
- [Appendix J: Canonical Conformance Bundle](spec/appendices/j-canonical-conformance-bundle.md)
- [Appendix K: Contributor Change Procedure](spec/appendices/k-contributor-change-procedure.md)

## Reviewing changes

Follow the six bounded milestones in [REVIEWING.md](REVIEWING.md). Keep each pull request focused on one milestone or one clearly named issue whenever possible.
