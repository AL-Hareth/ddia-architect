# Encoding, APIs, and schema evolution

Book: Ch 4, "Encoding and Evolution" (sections: Formats for Encoding Data; The merits of schemas; Modes of Data Flow)

## Core ideas

- During rolling deploys, and whenever clients update on their own schedule (mobile apps, partner integrations, devices), old and new code run at the same time. Data therefore needs:
  - **backward compatibility**: new code can read data written by old code, and
  - **forward compatibility**: old code can read data written by new code, ignoring what it doesn't understand.
- **Language-specific serialization** (Java Serializable, Python pickle, Ruby Marshal and the like) ties data to one language, handles versioning poorly, is often inefficient, and can let decoding instantiate arbitrary classes, which is a security hole. Keep it out of anything persisted or sent between processes.
- **JSON, XML, CSV**: universally supported and human-readable, but loose about types. Large integers lose precision in some parsers (JavaScript can't represent integers above 2^53 exactly, so send 64-bit IDs as strings), binary data needs base64, and schemas are optional. Fine for public APIs and data exchange, given a written compatibility policy.
- **Schema-driven binary formats** (Protocol Buffers, Thrift, Avro): compact; the schema doubles as documentation; compatibility can be checked mechanically; code can be generated.
  - Protobuf and Thrift identify fields by numeric *tags*. Fields may be renamed, but a tag must never be reused or reassigned. New fields must be optional or have defaults. Only optional fields may be removed, and their tags stay retired.
  - Avro has no tags. The reader resolves the *writer's schema* against its own *reader's schema* by field name, so it must know which schema wrote the data (embedded in the file, looked up by version in a schema registry, or agreed when a connection opens). This suits dynamically generated schemas, such as exporting database tables.
- **Three ways data flows between processes**, each with its own compatibility concerns:
  1. *Through databases*: data outlives code; a row written years ago is still read today. Old code may read a record written by new code, modify it, and write it back, silently dropping fields it didn't know about. Many databases can add a nullable column without rewriting existing rows.
  2. *Through services (REST and RPC)*: a remote call is not a local function call. It can time out with an unknown outcome, its latency varies wildly, retries can duplicate work unless requests are idempotent, and everything must be encoded. Servers usually upgrade before clients, so requests need backward compatibility and responses forward compatibility. Public APIs need explicit versioning.
  3. *Through asynchronous messages (brokers, actors)*: the broker buffers when consumers are slow, redelivers after crashes, decouples senders from receivers, and fans out to several consumers. Producers and consumers deploy independently, so message schemas need compatibility in both directions.

## Questions to ask

- Who produces and who consumes each kind of data? Do they deploy independently?
- Are there clients that can't be upgraded on demand (mobile, desktop, partners, devices)?
- How long is data kept: hours (messages) or years (database rows, archived events)?
- Is the API public or internal? How many teams consume it?
- How often does the schema change, and who approves changes?

## Decision guide

- **Public or external APIs**: JSON over HTTP with a documented schema (for example OpenAPI) and a versioning and deprecation policy.
- **Internal service-to-service calls across teams**: Protobuf/gRPC or similar, with compatibility checks in CI.
- **Event streams and long-lived logs**: a schema-registry-backed format (Avro or Protobuf) with the compatibility mode enforced when publishing.
- **Compatibility rules for any format**: add fields as optional with defaults; never reuse a field number or name with a different meaning; deprecate before removing; change types only in provably safe ways; preserve unknown fields through read-modify-write cycles.
- **Database schema changes**: prefer additive changes. For breaking ones, use expand and contract: add the new shape, write both, backfill, switch reads, and remove the old shape in a later deploy. Each step must be safe with old and new code running side by side, and each must have a rollback.
- **Every remote call** gets a timeout, a retry policy, and an idempotency strategy (see `distributed-failures.md`).

## Red flags

- Native language serialization for stored or transmitted data.
- 64-bit numeric IDs sent as JSON numbers to JavaScript clients.
- A breaking change (renamed or removed field, changed type) deployed with no window in which both versions work.
- Old code that rewrites records and drops fields it doesn't recognize.
- RPC calls treated like local calls: no timeouts, blind retries of non-idempotent operations.
- Events or messages with no schema at all, consumed by several teams.

## What to record

- The encoding format for each interface (public API, internal RPC, events, storage) and its compatibility mode.
- Who owns each schema, and how changes are reviewed and checked.
- The migration procedure for breaking schema changes, and the rollback path.
