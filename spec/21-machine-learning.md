# 21. Machine Learning

*Reserved for a future version of this specification.*\
ML/MLlib algorithm contracts are deferred from v1.0. Spark Connect currently exposes two ML proto entry points: `Command.ml_command` for operations such as Fit, Evaluate, Write, Read, and model lifecycle actions; and `Relation.ml_relation` for Transform and Fetch. Apache Spark implements these messages through Spark MLlib.\
A future specification version must define the intended portability boundary before adding ML to a Required Profile. That work may distinguish protocol and model-lifecycle interoperability from algorithm-specific numerical compatibility, and it must publish canonical rows and TCK cases for whichever contract the community adopts. Version 1.0 does not decide that future scope.\
A v1.0-conforming server MAY implement ML operations. Those operations remain outside `SC-1.0-P1` and do not affect its TCK result. The proto messages, including `MlCommand`, `MlRelation`, and per-operator parameter conventions, remain available in the pinned IDL; their presence does not add a v1.0 conformance requirement.
