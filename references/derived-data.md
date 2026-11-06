# Derived data: batch, streams, CDC, event sourcing

Book: Ch 10, "Batch Processing", and Ch 11, "Stream Processing" (sections: The output of batch workflows; Beyond MapReduce; Transmitting Event Streams; Databases and streams; Processing Streams; Fault tolerance)

## Core ideas

- **System of record vs derived data.** The system of record holds the authoritative version; it is written first and usually normalized. Derived data (caches, search indexes, materialized views, denormalized read models, analytics tables, ML features) is computed from it and can be recomputed if lost. Being explicit about which is which untangles most architectures.
- **Batch processing** (bounded input, a job that finishes): inputs are never modified, and outputs are derived and can feed further jobs. Because jobs have no side effects beyond their output, frameworks can retry failed tasks and discard partial output, so the result is as if nothing had failed. Write job output to a new location (new files, a freshly built index or key-value store) and switch over atomically. Avoid having a large job write row by row into a live production database: it overloads the database and leaves partial results behind on failure. Immutable inputs also make human mistakes recoverable: fix the bug and rerun, or roll back to the previous output. Modern dataflow engines avoid writing every intermediate step to disk and recover by recomputing, which works best when operators are deterministic.
- **Streams** (unbounded input, a job that never finishes): events flow through a messaging system.
  - Ask two questions of any messaging setup: what happens when producers outpace consumers (drop messages, buffer them, or apply backpressure), and what happens when a node crashes (can messages be lost)?
  - *Queue-style brokers* (AMQP/JMS-like) hand individual messages to consumers and delete them once acknowledged. Good for distributing tasks where order doesn't matter, each message is expensive to process, and nobody needs to re-read history. Redelivery after a crash can reorder messages.
  - *Log-based brokers* append messages to partitioned, durable logs. Within a consumer group, each partition is read in order by one consumer, which tracks its offset. Good for high throughput, per-key ordering, and replaying history to build new views. Parallelism is capped by the partition count, and one slow message holds up its whole partition.
- **Keeping different systems in sync**:
  - *Dual writes* (the application writes to the database, then to the search index, cache, or broker) are broken. Two concurrent clients can apply their writes in different orders to different systems, leaving them permanently inconsistent, and one write can succeed while the other fails.
  - *Change data capture (CDC)* treats the database's change log as a stream. The database is the leader and every derived system is a follower applying changes in the same order. A new consumer starts from a consistent snapshot plus the log position it corresponds to. *Log compaction* (keeping only the latest value per key) lets a topic act as a complete, rebuildable copy of the data.
  - *Event sourcing*: the application stores immutable domain events ("SeatReserved", "OrderCancelled") as the source of truth and derives current state from them. Distinguish *commands* (requests that may still be rejected after validation) from *events* (facts that have been accepted). Benefits: a natural audit trail, easier debugging and recovery from bugs, and several read models derived from the same facts. Costs: replay time (mitigated with snapshots), schema evolution for events that live forever, and genuine deletion (legal erasure) becomes hard.
- **Processing streams**: common uses are updating derived stores, alerting and pattern detection (complex event processing), windowed analytics, and maintaining materialized views.
  - *Event time vs processing time*: aggregate by when events happened, not when they arrived, or a consumer restart creates false spikes. Late (straggler) events need a policy: drop them, or emit corrections. Device clocks can't be trusted; recording the event time on the device, the send time on the device, and the receive time on the server makes it possible to estimate and correct the device's clock offset.
  - *Windows*: tumbling (fixed size, no overlap), hopping (fixed size, overlapping), sliding (events within some interval of each other), session (bursts of activity separated by gaps).
  - *Joins*: stream–stream (match events within a window, such as a search followed by a click), stream–table (enrich events from a local copy of a table kept current through CDC), table–table (maintain a materialized view of a join). Results depend on the order in which events are processed; make joins against changing data deterministic, for example by referencing specific versions of records.
- **Fault tolerance: "exactly once" really means "effectively once".** Processing may happen more than once; the *effect* must not. Techniques: microbatching or checkpointing for the processor's own state; atomically committing output together with input offsets inside the framework; or idempotent writes (store the input offset or event ID with the output and skip duplicates, with fencing against zombie processors). Side effects in external systems need idempotency keys.

## Questions to ask

- For every store: system of record or derived? If derived: from what, by which mechanism, with what maximum staleness? Can it be rebuilt from scratch, and how long would that take?
- What needs to react to changes, and how quickly (search, cache invalidation, notifications, analytics, other services)?
- Ordering: does it matter per entity, per user, globally, or not at all?
- What happens if a consumer processes a message twice? If it misses one?
- Would event sourcing pay for itself here (audit, history, multiple read models, long-running workflows), or is it overhead?
- For analytics on events: windows by event time? How late can events arrive, and what happens to late ones?

## Decision guide

- **One system of record per piece of data.** Everything else is derived through an ordered change log: CDC from the database, or a transactional outbox (the change and its event written in the same database transaction, then published from the outbox table). Never dual writes.
- **Broker choice**: log-based (Kafka, Redpanda, Pulsar, Kinesis and similar) for ordered, replayable streams that feed derived views and analytics; queue-based (RabbitMQ, SQS and similar) for distributing independent tasks. Partition by the entity whose events must stay in order.
- **Consumers**: idempotent; deduplicate on event ID or offset; store the offset atomically with the output where possible; able to rebuild their state by replaying from the beginning or from a snapshot.
- **Batch vs stream**: batch for full recomputation, backfills, heavy analytics, and model training; streams for low-latency derived views. Prefer a single processing code path where possible (reprocess history by replaying the log) over separate batch and streaming implementations of the same logic, which drift apart.
- **Batch outputs** go to a new location and are swapped in atomically, keeping the previous output for rollback.
- **Event sourcing**: adopt it where history is a first-class requirement (finance, compliance, collaborative workflows, multiple projections). Plan snapshots, event schema evolution (`encoding-and-evolution.md`), and deletion (for example per-person encryption keys that can be destroyed) up front.
- **Time**: window by event time with an explicit lateness policy, and decide which clock is trusted.

Examples to verify against current documentation: Debezium for CDC; Flink, Kafka Streams, or Spark Structured Streaming for stream processing; Spark, dbt, or warehouse SQL for batch.

## Red flags

- The application writing the same change to several systems (dual writes).
- Cache or search-index updates with no single ordered source, or invalidation scattered across many code paths.
- Exactly-once *delivery* assumed; consumers that aren't idempotent.
- Derived stores that can't be rebuilt from the source, or nobody knows how long a rebuild takes.
- Large batch jobs writing directly into the production OLTP database.
- Processing-time windows where event time matters; no policy for late events.
- Two implementations (batch and streaming) of the same business logic.
- Event sourcing adopted without a plan for deletion, snapshots, and event versioning.

## What to record

- A dataflow diagram: the write path from the system of record to every derived store, with the mechanism, ordering key, and staleness of each.
- The delivery semantics and idempotency mechanism of each consumer.
- How each derived store is rebuilt, and how long that takes.
- Event schemas, their compatibility mode, and retention.
