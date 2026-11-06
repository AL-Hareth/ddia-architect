# Replication

Book: Ch 5, "Replication" (sections: Leaders and Followers; Problems With Replication Lag; Multi-Leader Replication; Leaderless Replication)

## Why replicate? Ask which reasons apply, because they lead to different designs

- **High availability**: keep working when a machine, zone, or datacenter fails.
- **Read scaling**: serve more reads than one machine can.
- **Latency**: keep data near users who are spread geographically.
- **Disconnected operation**: keep working offline and sync later.

## Core ideas

**Single-leader**: all writes go through one leader; followers apply its stream of changes; reads may go to any replica.
- *Synchronous* replication guarantees a follower has the data before the write is acknowledged, but a slow or dead follower blocks writes. *Asynchronous* replication is fast and never blocks, but a leader failure can lose recently acknowledged writes. A common middle ground is one synchronous follower plus asynchronous others.
- *A new follower* starts from a consistent snapshot, then catches up from the exact log position that snapshot corresponds to.
- *Failover* (detect that the leader is dead by timeout, choose a new one, redirect writes) is full of hazards. Writes the old leader never replicated get discarded. If other systems already acted on those writes, IDs can be reused and data can leak between records. Two nodes can both believe they are the leader (split brain). A timeout that is too short triggers needless failovers under load; one that is too long prolongs outages. This is why many teams prefer human-approved failover.
- *Replication log formats*: statement-based (breaks on nondeterministic functions such as NOW() or RAND()), write-ahead-log shipping (tied to the storage engine's on-disk format, which complicates upgrades), logical or row-based (decoupled from the engine and usable for change data capture), trigger-based (flexible but with more overhead).

**Replication lag**: with asynchronous followers, reads from followers are only *eventually* consistent, and lag can stretch to seconds or minutes under load or during failures. Three guarantees an application often needs:
- *Read-your-writes*: users always see changes they submitted themselves. Read anything the user can edit from the leader, or read from the leader for a while after the user writes, or read from a replica known to have caught up to the user's last write position. Making this hold across a user's devices requires storing that position centrally.
- *Monotonic reads*: a user never sees time go backwards. Route each user's reads to the same replica.
- *Consistent prefix reads*: causally related writes appear in order, so an answer never shows up before its question. This gets harder once data is partitioned.
- Ask "what happens if lag grows to minutes?" If the honest answer is "bugs", the application needs stronger guarantees, not hope.

**Multi-leader**: several nodes accept writes and replicate to each other.
- Use cases: a leader per datacenter (local write latency, surviving a datacenter outage), offline-capable clients (each device acts as a leader), collaborative editing.
- The central problem is *write conflicts*. The options: avoid conflicts by routing every write for a given record to the same leader (a home region per user or tenant); converge with last-write-wins (simple, but silently discards writes); merge the values; keep all versions and let the application or the user resolve them; or use data types designed to merge (CRDTs, operational transformation).
- Auto-increment IDs, triggers, and integrity constraints become hazardous. Ring and star topologies break when one node fails; all-to-all topologies can deliver causally related writes out of order unless version vectors are used.

**Leaderless (Dynamo-style)**: clients write to and read from several replicas in parallel.
- Quorums: with *n* replicas, a write is confirmed by *w* of them and a read queries *r*; if *w + r > n*, a read normally reaches at least one node with the latest write. Stale replicas are repaired on read and by background anti-entropy.
- Even with *w + r > n*, edge cases mean this is **not** linearizable: concurrent writes, a write concurrent with a read, a write that failed on some nodes but isn't rolled back on the others, a failed node restored from a stale replica, sloppy quorums with hinted handoff. Treat it as eventual consistency with a tunable probability of freshness.
- Concurrent writes need a happened-before check (version vectors) and a merge strategy. Last-write-wins loses data unless each key is written only once.

## Questions to ask

- Which of the four reasons apply? What is the recovery point: may the last few seconds of acknowledged writes be lost?
- After a user writes, must they see it immediately? On their other devices too?
- Would it cause harm if a user's reads occasionally went backwards in time?
- Are writes needed in several regions, or is one write region with local read replicas enough?
- Must clients work offline?
- Will failover be automatic or human-approved, and how is the old leader stopped from accepting writes?

## Decision guide

- **Default**: single-leader with managed failover; one synchronous (or semi-synchronous) follower if acknowledged writes must never be lost; asynchronous read replicas as needed.
- **Read scaling**: add replicas only together with explicit read-your-writes and monotonic-read handling, or use caches with a clear invalidation path.
- **Multi-region**: first try one write region with read replicas near users. If local writes are truly needed, give each user, tenant, or record a home region so conflicts can't arise. Accept true multi-leader writes to the same records only with a designed conflict-resolution strategy (CRDTs or explicit merging), never last-write-wins by default.
- **Offline clients**: treat every device as a leader, with per-record version vectors or CRDTs and a sync protocol that preserves conflicting versions instead of overwriting them.
- **Leaderless**: for very high write availability where stale reads and merges are acceptable. Budget for monitoring staleness, which is harder than in single-leader systems.
- **Zero data loss**: synchronous or consensus-based replication (see `consistency-and-consensus.md`), accepting the added write latency.

Examples to verify against current documentation: PostgreSQL or MySQL streaming replication (single-leader); managed databases with multi-zone failover; Cassandra/ScyllaDB (leaderless); CouchDB-style sync, or CRDT libraries such as Automerge and Yjs, for offline and collaborative use.

## Red flags

- Reads from asynchronous replicas with no read-your-writes or monotonic-read handling.
- Automatic failover with no fencing of the old leader.
- Last-write-wins on data that can be modified concurrently, especially with wall-clock timestamps.
- Multi-leader chosen "for availability" without a conflict-resolution design.
- Quorum reads treated as linearizable.
- Promoting a lagging follower after caches, search indexes, or external services have already seen IDs or data from the old leader.
- No monitoring of replication lag.

## What to record

- Why each replica exists, the topology, and synchronous vs asynchronous.
- The failover policy: detection, decision (automatic or human), and how the old leader is fenced.
- The guarantee on each read path (read-your-writes, monotonic reads, consistent prefix) and how it is achieved.
- The conflict-resolution strategy wherever there are multiple writers.
- Replication-lag metrics and alert thresholds.
