# K. Contributor Change Procedure (Informative SOP)

This appendix describes how contributors coordinate a Spark Connect change. It does not define conformance. Section 6.1.1 assigns authority to each artifact, §22.2 defines protocol evolution, and §22.1.3 is the sole finalization checklist. Those sections control if this procedure conflicts with normative text.\
Reviewers assess this procedure in Milestone 6 together with the TCK, publication, and evolution rules. After acceptance, the Apache Spark repository should maintain the operational copy at sql/connect/docs/protocol-change-process.md and link it from the Connect documentation, the protobuf README, and the TCK repository.

## K.1 Start with a change record

Before editing an artifact, record:

* the observable behavior being added, corrected, removed, or clarified;\
* the affected specification and profile versions;\
* whether the change is editorial, an implementation correction, additive, or incompatible;\
* the affected protobuf messages, specification sections, canonical row IDs, clients, server components, tests, and repositories;\
* the expected compatibility and version-skew behavior; and\
* links to every related issue, document revision, and pull request.

Use these change classes:

* **Editorial:** changes presentation, organization, spelling, or links without changing wire shape, profile membership, observable behavior, or a TCK assertion.\
* **Implementation correction:** changes code or implementation tests to satisfy an unchanged contract. If the contract must change, reclassify the change as additive or incompatible.\
* **Additive:** introduces a new wire element, behavior, profile row, or test obligation without invalidating an existing contract. Apply the minor-version and protocol-evolution rules in §22.\
* **Incompatible:** removes or changes an accepted wire shape, required behavior, or existing profile meaning. Apply the major-version and approval rules in §22.

Wire compatibility alone does not place a feature in the Required Profile. Resolve an unclear classification before publishing a profile or conformance claim.

## K.2 Identify the controlling and affected artifacts

Apply §6.1.1 to each part of the change. Record which artifact controls wire shape, profile membership, observable semantics, and conformance evidence. Update the controlling artifact rather than relying on an implementation, generated file, short-draft summary, or TCK case to create a rule.\
Check each of the following and mark an item not applicable only with a reason:

* protobuf IDL and field comments;\
* generated client stubs;\
* reference server and affected clients;\
* full normative specification, short review draft, and active milestone review packets;\
* canonical bundle, manifests, and generated mirrors;\
* core TCK, adapter metadata, and result reporting;\
* source-validation evidence; and\
* version identifiers, revision history, commit pins, paths, and digests.

## K.3 Edit protobufs and regenerate derived code

For a wire change:

1. Edit the applicable files under `sql/connect/common/src/main/protobuf/spark/connect/`.\
2. Follow `sql/connect/docs/adding-proto-messages.md` for required and optional field semantics.\
3. State absence and default behavior explicitly where they differ.\
4. Check field numbers, field types, enum values, presence, and `oneof` changes against §22.2.\
5. Define handling for unknown fields, enum values, and selected alternatives where the portable contract requires it.\
6. Follow `sql/connect/common/src/main/protobuf/README.md` and run `dev/connect-gen-protos.sh` by one of its documented methods.\
7. Run any additional repository-prescribed generators for affected clients.\
8. Review the generated changes and confirm that they result only from the IDL update.

Do not edit generated stubs by hand. A protobuf comment documents a wire field; it does not by itself define Required Profile membership or portable behavior.

## K.4 Update implementation, specification, and profile artifacts

Update the affected artifacts as one coordinated change:

* Put portable normative semantics in the full specification.\
* Update the canonical bundle and its manifest members when membership, constraints, mappings, schemas, or negative cases change.\
* Preserve existing canonical row IDs; assign new IDs to new requirements rather than silently changing an old ID's meaning.\
* Update the reference server and affected clients.\
* Update the short draft and any affected active milestone review packets with the same draft number, date, change summary, milestone placement, and controlling-section links.\
* Regenerate informative appendices and other bundle mirrors after changing the bundle.

The short draft and active milestone review packets are review aids. They must not create a requirement absent from the full specification or canonical bundle. Appendix J controls bundle construction and serialization.

## K.5 Update tests, compatibility checks, and evidence

For each changed in-profile rule:

* add or update positive and negative TCK cases;\
* cite the controlling canonical row IDs and normative sections;\
* add regression coverage for existing behavior affected by the change;\
* update the §22.6 source-validation evidence required by §22.1.3; and\
* add focused implementation tests for the positive path, invalid input, error transport, and unchanged behavior at risk.

For a wire or behavioral change, record the expected and observed result for each applicable version-skew case:

* the previous conforming client against the previous server as a baseline;\
* the previous conforming client against the changed server;\
* the changed client against the previous server;\
* the changed client against the changed server when it claims the new profile;\
* unknown-field, unknown-enum, and unknown-`oneof` handling; and\
* use of the new feature against an endpoint that does not claim the new profile.

Distinguish successful protobuf decoding from feature support and conformance. An implementation test may support source validation, but it does not replace a required TCK case.

## K.6 Coordinate review and publication

Cross-link the Apache Spark, full specification, short draft, active milestone packet, TCK, client, and dialect-profile changes. Each pull request identifies its dependencies, the shared change record, and any provisional pins.\
Use this publication sequence:

1. Review the coordinated changes and run tests with provisional identifiers.\
2. Land the Apache Spark artifacts that establish the immutable publication commit.\
3. Compute the canonical bundle SHA-256 from the exact bytes at that commit.\
4. Replace provisional values in the full specification, short draft, active milestone packets, TCK metadata, and validation evidence with the immutable commit, path, and digest.\
5. Rerun the affected TCK and compatibility checks against the pinned artifacts.\
6. Apply the finalization gate in §22.1.3.

Do not cite a branch, moving tag, working-tree file, or digest computed from bytes different from the committed bundle.

## K.7 Complete the change record

The record is complete when every affected artifact has a revision or pull-request link, every applicable test and validation result is recorded, all provisional pins are resolved, and every remaining discrepancy has an owner and release-blocking status.\
This completion record supports review. It does not replace §22.1.3 or create another finalization gate.
