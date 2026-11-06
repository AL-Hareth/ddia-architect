# End-to-end correctness, verification, operations, and ethics

Book: Ch 12, "The Future of Data Systems" (in the final 2017 edition; sections: Data Integration; Unbundling Databases; Aiming for Correctness; Doing the Right Thing), plus the maintainability part of Ch 1.

## Core ideas

- **No single tool does everything.** Real applications combine a database, a cache, a search index, an analytics store, and more. Integration works best when the dataflow is explicit: one ordered log of changes from which each specialized store is derived. When a change has to reach several systems, ordered asynchronous propagation with idempotent consumers is more robust than distributed transactions, because a fault in one consumer stays contained instead of blocking everything else.
- **Write path vs read path.** Caches, indexes, and materialized views move work from read time to write time. Where to draw that boundary is a design choice made per use case. The same idea extends to clients: instead of polling, they can subscribe to changes and keep their local state current.
- **Evolving live systems.** With derived views, migrations can be gradual: build the new view alongside the old one from the same log, shift traffic over a little at a time, and keep the old view as a rollback path until confidence is high.
- **The end-to-end argument.** Guarantees from lower layers (TCP's deduplication, a database transaction) don't add up to correctness for the application. A user who double-clicks "pay", or a client that retries after a timeout, sends what every lower layer sees as a brand-new request. Duplicate suppression therefore has to be end to end: generate a unique request ID at the edge (the client, or the first service) and carry it all the way to the system of record, where a uniqueness constraint makes the operation take effect only once.
- **Enforcing constraints without distributed transactions.** A uniqueness check needs a single decision point. In a log-based design, route every request about the same key (username, seat, account) to the same log partition and let one consumer process them sequentially; it decides deterministically which request wins. An operation that touches several partitions (a transfer between two accounts) can be recorded as a single request event carrying a request ID, from which the debit and the credit are derived and applied idempotently, with no atomic commit needed.
- **Integrity vs timeliness.** "Consistency" mixes up two things. *Timeliness* means users see an up-to-date state; violations look like stale reads and fix themselves with time. *Integrity* means no corruption, no loss, no contradictions; violations are permanent until someone repairs the data. Most applications can relax timeliness (eventual consistency) but never integrity. Integrity can be preserved without synchronous coordination when every write is a single immutable message, derived state is computed deterministically, request IDs flow end to end, and messages can be reprocessed.
- **Loosely interpreted constraints.** Many business rules may be violated briefly and then corrected, because the business already has a process for it (overbooking, backorders, refunds, apologies). Compare the cost of an occasional apology with the cost of coordinating on every request, and choose deliberately.
- **Trust, but verify.** Hardware and software, databases included, occasionally lose or corrupt data. Build in self-checks: reconcile derived stores against the system of record, verify checksums, read back what was written, and test that backups actually restore. Systems built on immutable event logs are much easier to audit.
- **Doing the right thing.** Data is about people. Collect only what is needed, be clear about purpose and consent, plan retention and deletion, and consider how predictive or automated decisions could be biased, unaccountable, or self-reinforcing. Stored personal data is a liability as well as an asset.
- **Operability** (Ch 1): visibility into the system's health, automation with manual override, good documentation, predictable behavior, no dependence on any single machine, and safe defaults.

## Questions to ask

- Where is the request ID or idempotency key generated, and is it enforced at the system of record?
- Which invariants are about integrity (must never be violated) and which about timeliness (may lag briefly)? Which could be handled by detecting and compensating?
- How would silent data loss or corruption be noticed? Is there a reconciliation job? When was a backup last restored?
- What are the service-level indicators (from the load parameters and latency targets), and what alerts fire on them? Are replication lag and consumer lag monitored?
- What personal data is stored, why, for how long, and how is it deleted everywhere (logs, backups, derived stores, analytics)?
- How are schema changes and the reprocessing of derived views rolled out, and rolled back?

## Decision guide

- **Idempotency end to end**: the client or edge service generates a request ID; the system of record stores it under a unique constraint; every downstream consumer deduplicates on it; external calls (payments, emails) pass it along wherever the provider supports idempotency keys; after a timeout, look up the outcome by ID instead of re-executing.
- **Classify every constraint**: hard integrity constraints are enforced synchronously at a single decision point (one database transaction, or one log partition). Soft ones are checked asynchronously and compensated.
- **Integration**: prefer log-based propagation with idempotent consumers over distributed transactions.
- **Verification**: scheduled reconciliation between the system of record and each derived store, end-to-end checks on critical flows, and regular restore drills.
- **Observability from day one**: dashboards and alerts for the load parameters, p50/p99/p999 of each key operation, error rates, replication lag, consumer lag, and queue depth. Alert on error-budget burn rather than on single spikes.
- **Safe change**: gradual rollouts, feature flags, reversible migrations, the ability to rebuild derived data, and runbooks for the failure modes listed in the plan.
- **Privacy**: data minimization, retention periods, a deletion path that reaches logs, backups, and derived stores, and access control and auditing for personal data.

## Red flags

- No request IDs, so a client retry or a double-click can charge, book, or send twice.
- Relying on database transactions alone for an operation that spans a client retry or an external service.
- "Exactly once" claimed for a pipeline whose sink isn't idempotent.
- No way to detect that a derived store has drifted from the system of record.
- Backups that have never been restored.
- Personal data copied into logs, analytics, and event streams with no deletion plan.
- Observability planned "for later".

## What to record

- The idempotency design, from the client through the system of record to every downstream consumer.
- A constraint classification table: integrity or timeliness, enforcement point, compensation.
- The verification plan: reconciliation, checksums, restore drills.
- Service-level indicators, objectives, and alerts; runbooks for the top failure modes.
- The data-protection plan: which personal data, for what purpose, retention, and the deletion path.
