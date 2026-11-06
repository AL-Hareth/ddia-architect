# Red-flag review

Use this at Gate 2 and as the backbone of a review of an existing design. For every item that applies, either change the design or record in the plan why the risk is accepted. Report the hits to the developer ordered by severity, each with its fix.

**Severity**
- **Critical**: can silently lose or corrupt data, or violate a must-never invariant.
- **High**: an outage or widespread incorrect behavior under plausible faults or load.
- **Medium**: performance, cost, or operability problems at the expected scale.
- **Low**: maintainability and future flexibility.

Each item reads: the smell, why it hurts, then the usual fix.

## Requirements and scope
- **No numbers.** Load and targets described as "fast" or "scalable" can't be checked against anything. → Back-of-envelope estimate with marked assumptions (`requirements-and-load.md`). *(Medium)*
- **Averages as targets.** Means hide the tail, which is where the heaviest users and fan-out pain live. → Percentiles per key operation, measured at the client. *(Medium)*
- **Distribution without a reason.** Sharding, microservices, or multi-region for a workload one node handles adds permanent complexity and new failure modes for no benefit. → One node plus replicas; record the trigger for scaling out. *(Medium)*
- **No one owns operations.** Nobody to watch, upgrade, and recover the system means faults become outages. → Name the operators; prefer managed services for a small team. *(Medium)*

## Data model and storage
- **Document store for highly relational data.** Joins move into application code and duplicated data drifts. → Relational, or relational with JSON columns. *(Medium)*
- **Shared mutable facts copied into many places with no sync mechanism.** Copies silently disagree after the first update. → Normalize, or derive the copies through CDC. *(High)*
- **Heavy analytics on the OLTP primary.** Large scans steal resources from user-facing requests. → Reporting replica or a separate analytical store. *(High)*
- **Write-heavy store without compaction or vacuum headroom.** Background maintenance falls behind, tail latency climbs, disks fill. → Capacity for write amplification; monitor compaction backlog. *(Medium)*
- **In-memory store holding the only copy of important data.** A restart loses it. → Configure and test persistence and replication, or treat it as a rebuildable cache. *(Critical)*

## Encoding and evolution
- **Native language serialization for stored or transmitted data.** Locks data to one language, versions badly, and can allow code execution on decode. → A schema-defined format. *(High)*
- **Breaking schema change with no compatible window.** During a rolling deploy, old and new code can't read each other's data. → Expand and contract; compatibility checks in CI. *(High)*
- **Old code drops unknown fields on read-modify-write.** New data is silently erased by old code during rollouts. → Preserve unknown fields; roll out readers before writers. *(Critical)*
- **64-bit IDs as JSON numbers for JavaScript clients.** IDs above 2^53 get corrupted. → Send them as strings. *(Medium)*
- **Unschematized events shared between teams.** Any producer change can break consumers unnoticed. → Schema registry with an enforced compatibility mode. *(Medium)*

## Replication
- **Reads from asynchronous replicas with no lag handling.** Users don't see their own writes, or see data jump backwards. → Read-your-writes and monotonic-read routing. *(Medium)*
- **Automatic failover without fencing.** Two leaders accept writes (split brain) and data diverges or is lost. → Consensus-based leader election plus fencing, or human-approved failover. *(Critical)*
- **Last-write-wins with wall-clock timestamps on concurrently modified data.** Writes are discarded silently, depending on clock skew. → Avoid conflicts by routing, or merge them with version vectors or CRDTs. *(Critical)*
- **Multi-leader without a conflict-resolution design.** Conflicts are inevitable; without a plan they are resolved arbitrarily. → A home region per record, or explicit merge semantics. *(Critical)*
- **Quorum reads treated as linearizable.** Edge cases return stale data while the system claims freshness. → Linearizable store for the operations that need it. *(High)*
- **Asynchronous replication where acknowledged writes must survive.** A leader failure loses acknowledged data. → A synchronous or consensus-replicated copy. *(Critical)*
- **No replication-lag monitoring.** Lag grows unnoticed until it causes visible bugs. → Lag metrics with alerts. *(Medium)*

## Partitioning
- **hash(key) mod N.** Adding or removing a node moves almost all data. → Many fixed partitions, or dynamic splitting. *(High)*
- **Sequential key leading a range-partitioned, write-heavy table.** Every new write lands on one partition. → Hash or prefix the key. *(High)*
- **Known hot keys with no plan.** One partition saturates while the rest idle. → Key splitting, a dedicated partition, caching, or an admission queue. *(High)*
- **Scatter/gather queries on a latency-critical path.** The slowest partition sets the response time. → A global index or a derived search store. *(Medium)*
- **Invariants spanning partitions without a mechanism.** Concurrent requests violate them. → Re-key so the invariant lives in one partition, or route decisions through a single log partition. *(Critical)*
- **Partition count fixed too low for growth.** Partitions become unmanageably large and can't be spread out. → Size for three to five years. *(Medium)*

## Transactions and concurrency
- **Read-modify-write without an atomic operation, version check, or lock.** Concurrent updates are lost. → Atomic statement, compare-and-set, or row lock. *(Critical)*
- **Check-then-act invariant under read committed or snapshot isolation.** Write skew lets two transactions jointly break the rule. → Serializable isolation, a constraint, explicit locks, or a materialized conflict. *(Critical)*
- **Trusting isolation-level names.** "Repeatable read" means different things in different databases. → Verify the actual guarantees for the version in use. *(High)*
- **Blind retries of non-idempotent operations.** A lost acknowledgment plus a retry means double execution. → Idempotency keys enforced by a unique constraint. *(Critical)*
- **Transactions spanning network calls or user think-time.** Locks are held for long, unpredictable periods. → Short transactions; external calls outside them. *(Medium)*

## Distributed-systems hazards
- **Wall-clock ordering across machines.** Clock skew reorders events and drops writes. → Logical clocks or a single sequencer. *(Critical)*
- **Time-of-day clock for durations.** Clock jumps produce negative or huge durations and premature timeouts. → Monotonic clocks. *(Medium)*
- **Locks or leases without fencing tokens.** A paused former holder acts after losing the lock. → Fencing tokens checked by the resource. *(Critical)*
- **Timeout treated as definite failure.** The operation may have succeeded; compensating or retrying blindly duplicates or corrupts. → Look up the outcome by ID; idempotent retries. *(High)*
- **Retries without limits, backoff, or jitter.** Retry storms turn a blip into an outage. → Bounded retries with exponential backoff and jitter, circuit breakers. *(High)*
- **No defined behavior during a network partition.** The system does something accidental and untested. → Choose per operation: fail, or proceed and reconcile. *(High)*
- **Home-grown consensus, leader election, or distributed locking.** Subtly wrong under real network conditions. → etcd, ZooKeeper, or the database's own mechanisms. *(Critical)*

## Consistency and coordination
- **Uniqueness or limits enforced by check-then-insert against an eventually consistent store or replica.** Duplicates and oversells under concurrency. → A unique constraint in one database, or one log partition per key. *(Critical)*
- **XA/2PC across heterogeneous systems on the request path.** A coordinator failure blocks everything with locks held. → One log as the decision point, idempotent consumers, compensating actions. *(High)*
- **Cross-channel race.** Data written to store A, a message sent through B, and the consumer reads a lagging copy of A. → Read the exact referenced version, or use one ordered channel. *(High)*
- **Strong consistency everywhere, or nowhere.** Either needless latency and fragility, or hidden correctness bugs. → A per-operation consistency table. *(Medium)*

## Derived data and pipelines
- **Dual writes.** Races and partial failures leave systems permanently inconsistent. → CDC or a transactional outbox. *(Critical)*
- **Non-idempotent consumers; exactly-once delivery assumed.** Redelivery double-applies effects. → Deduplicate on event ID or offset; atomic offset-plus-output commits. *(Critical)*
- **Derived stores that can't be rebuilt, or unknown rebuild time.** A bug or corruption becomes unrecoverable. → Keep the source log or snapshots; rehearse a rebuild. *(High)*
- **Batch jobs writing row by row into the production OLTP database.** Overload, and partial results on failure. → Build output separately and swap it in atomically. *(High)*
- **Processing-time windows where event time matters.** Restarts and delays produce wrong numbers. → Event-time windows with a lateness policy. *(Medium)*
- **The same logic implemented in both batch and streaming code.** The two drift apart. → One code path; reprocess by replaying the log. *(Medium)*
- **Event sourcing without snapshots, event versioning, or a deletion plan.** Slow recovery, unreadable old events, legal exposure. → Plan all three up front. *(High)*

## End-to-end correctness and operations
- **No end-to-end request IDs.** Client retries and double-clicks charge, book, or send twice. → Request ID generated at the edge, enforced at the system of record. *(Critical)*
- **No reconciliation and untested backups.** Silent corruption and failed restores are discovered when it's too late. → Scheduled reconciliation; regular restore drills. *(High)*
- **No observability for load, percentiles, and lag.** Problems are found by users first. → Service-level indicators and alerts from the first slice. *(High)*
- **Personal data without a retention and deletion path.** Legal and ethical exposure that grows with every copy. → Minimize, set retention, and design deletion across all stores. *(High)*
- **Irreversible migration without backfill and rollback plans.** A bad migration can't be undone. → Expand and contract; keep the old path until the new one is verified. *(High)*
