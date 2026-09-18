# 13. Artifacts

*Reserved for a future version of this specification.*\
This chapter will define the `AddArtifacts` and `ArtifactStatus` RPCs and the artifact upload protocol — per-session storage for JAR, Python file, archive, and class-file artifacts, addressed under `jars/`, `pyfiles/`, `archives/`, and `classes/` paths, with CRC32 checksum validation.\
Artifacts are deferred from v1.0 because their primary use case is delivering dependencies to UDFs (Chapter 18), which is also deferred. Without UDFs in scope, the standalone use case for artifacts is narrow (primarily `CommonInlineUserDefinedDataSource`).\
A v1.0-conforming server MAY implement AddArtifacts and ArtifactStatus or return gRPC UNIMPLEMENTED for these RPCs. Such an implementation is outside SC-1.0-P1; the v1.0 specification makes no portable addressing, content, or future-compatibility claim for it beyond the pinned IDL wire shape.\
---
