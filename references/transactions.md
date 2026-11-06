# Transactions and isolation

Book: Ch 7, "Transactions" (sections: The Meaning of ACID; Single-Object and Multi-Object Operations; Weak Isolation Levels; Preventing Lost Updates; Preventing Write Skew and Phantoms; Serializability)

## Core ideas

- A transaction turns a messy set of partial failures and concurrency races into one outcome, commit or abort, that the application can simply retry. Transactions are a design choice with costs, not a law of nature. Without them, every failure mode must be handled by hand, and denormalized data quietly drifts out of sync.
- **ACID, precisely**:
  - *Atomicity* means abortability: on any fault, all of the transaction's writes are discarded.
  - *Consistency* means the application's invariants hold. That is the application's job; the database can only help, through constraints.
  - *Isolation* means concurrent transactions don't see each other's partial work. Ideally serializable; in practice often weaker.
  - *Durability* means committed data survives crashes (via disk, replication, or both). It is never absolute.
  - "ACID compliant" says little on its own. Ask which isolation level is actually provided.
- Multi-object transactions are needed for foreign-key integrity, for updating denormalized data, and for keeping secondary indexes in step with the data.
- **Retrying aborted transactions** is the point, with caveats: the commit may have succeeded and only the acknowledgment was lost (a retry then duplicates the work unless it is deduplicated); retrying under overload makes the overload worse (limit retries and back off); retry only transient errors such as deadlocks and serialization failures; side effects outside the database (emails, payments) happen even if the transaction later aborts.

### The anomalies, and what prevents each

| Anomaly | What happens | Prevented by |
|---|---|---|
| Dirty read | Seeing another transaction's uncommitted writes | Read committed and stronger |
| Dirty write | Overwriting another transaction's uncommitted write | Almost every database |
| Read skew (non-repeatable read) | Seeing different parts of the data at different points in time, so a backup or report mixes before and after states | Snapshot isolation (MVCC) |
| Lost update | Two read-modify-write cycles race and one write disappears | Atomic updates, explicit locks, compare-and-set, or automatic detection in some snapshot-isolation implementations |
| Write skew | Two transactions read the same data, decide based on it, then write *different* records; together they break an invariant | Serializable isolation only (or explicit locks on what was read) |
| Phantom | A write changes the result of a search another transaction already relied on | Serializable isolation; index-range or predicate locks |

Classic write-skew situations: at least one doctor must stay on call; no double-booking of a room or seat; usernames must be unique; an account must not go negative across several operations; two players can't move to the same square. The pattern is always *check a condition, then write based on it*.

### Isolation levels in practice

- **Read committed**: no dirty reads or dirty writes. A common default (PostgreSQL's, for example).
- **Snapshot isolation**: each transaction reads from a consistent snapshot, which is essential for long reads such as backups, reports, and integrity checks. Naming is a mess: several databases call it "repeatable read", Oracle calls it "serializable", and implementations differ. The 2017 edition of the book notes that PostgreSQL's repeatable read, Oracle's serializable, and SQL Server's snapshot isolation detect lost updates automatically, while MySQL/InnoDB's repeatable read does not. Verify this for the exact version in use.
- **Serializable**, implemented one of three ways:
  - *Actual serial execution*: one thread runs whole transactions, typically as stored procedures. Works when every transaction is short, the data mostly fits in memory, and the write throughput fits on one CPU core (or the data is partitioned so that transactions never span partitions).
  - *Two-phase locking (2PL)*: readers and writers block each other. Throughput suffers, tail latency suffers more, and deadlocks occur.
  - *Serializable snapshot isolation (SSI)*: optimistic. Transactions run against snapshots and are aborted at commit if a conflict made the outcome non-serializable. Performs well under low contention; aborts rise as contention rises. PostgreSQL's SERIALIZABLE level uses SSI.

## Questions to ask

- List every invariant that spans more than one record or more than one step. For each: which reads does the decision depend on, and which writes follow?
- Which operations are read-modify-write? Can each be expressed as a single atomic statement?
- How contended are the hottest records (a few seconds of frenzy on one item, or updates scattered thinly)?
- What does the client do when a commit times out?
- Are there long-running reads (exports, reports, backups) that need a consistent snapshot?

## Decision guide

- **Use a database with real multi-object transactions as the system of record**, unless there is a strong reason not to.
- **Choose per operation**, not globally:
  - Simple single-record writes: read committed plus atomic operations (`UPDATE … SET n = n + 1`).
  - Consistent multi-record reads: snapshot isolation.
  - Check-then-write invariants: serializable isolation (SSI where available), or explicit locks on the rows that were read (`SELECT … FOR UPDATE`), or a database constraint.
- **Prefer declarative constraints** whenever they express the invariant: UNIQUE, foreign keys, CHECK, exclusion constraints (for example, no overlapping time ranges for the same room). They are the cheapest correct mechanism.
- **Lost-update prevention, in order of preference**: a single atomic statement; a conditional update or compare-and-set on a version column; explicit row locks.
- **Claiming several things at once** (several seats, several items): one conditional statement that claims all of them only if all are free, rolled back unless the affected row count matches, is simple and correct.
- **When the invariant is about the *absence* of rows** ("no booking exists for this slot"), locking existing rows doesn't help. Use serializable isolation, a uniqueness or exclusion constraint, or as a last resort *materialize the conflict*: pre-create one lock row per room per time slot and lock that.
- **Retries**: retry serialization failures and deadlocks with a limit and backoff, and make the request idempotent end to end (an idempotency key under a unique constraint) so a retry after a lost acknowledgment does no harm.
- Keep transactions short. Never hold one open across user think-time or a network call to another service.

## Red flags

- Read-modify-write in application code (read a value, compute, write it back) with no atomic operation, lock, or version check.
- Check-then-act invariants running under read committed or snapshot isolation (write skew).
- Trusting the name of an isolation level without checking what it guarantees in this database version.
- Blind retries of non-idempotent operations.
- Transactions spanning external calls or user interaction.
- A store without multi-object transactions holding data that must stay mutually consistent.

## What to record

- An invariants table: invariant, enforcement mechanism (constraint, isolation level, lock, single-partition serial processing), and what happens when a violating request arrives.
- The isolation level for each key operation.
- The retry and idempotency policy.
