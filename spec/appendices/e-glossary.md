# E. Glossary

* **Session** — A logical execution context identified by `session_id`. Carries configuration, registered artifacts, and temporary views.\
* **Execution** — A single `ExecutePlan` invocation, identified by `operation_id`.\
* **Plan** — A `Plan` message carrying either a `Relation` (query) or `Command` (side effect).\
* **Relation** — A logical query operator (e.g., `Project`, `Filter`, `Join`); a node in a query plan tree.\
* **Expression** — A scalar sub-tree within a `Relation` (e.g., `Literal`, `Cast`, `UnresolvedFunction`).\
* **Command** — A side-effecting operation (e.g., `SqlCommand`, `WriteOperation`).\
* **Artifact** — A client-uploaded file (JAR, Python file, archive, classfile) made available within a session.\
* **Reference implementation** — The Apache Spark release named in §1.2 whose observable behavior is normative wherever this specification defers.\
* **Conformance** — The property of an implementation passing the Spark Connect TCK at a specific specification version.\
* **Wire extension — A feature that transmits a nonstandard payload through an explicit google.protobuf.Any extension point and a separately named payload type. A client convenience API is an out-of-profile public client API that lowers entirely to standard plans and RPCs; it is not a wire extension and requires no Any payload.**\
* **Reattachable execution** — An execution that the client may resume after a network disconnect via `ReattachExecute`.
