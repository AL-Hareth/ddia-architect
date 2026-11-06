# Consistency, ordering, and consensus

Book: Ch 9, "Consistency and Consensus" (sections: Linearizability; Ordering Guarantees; Distributed Transactions and Consensus; Membership and coordination services)

## Core ideas

- **Eventual consistency** only promises that replicas converge once writes stop: no bound on when, and no promise about what a read sees in the meantime. Stronger guarantees cost performance and, during network faults, availability.
- **Linearizability**: the system behaves as if there were a single copy of the data and every operation took effect atomically at one instant; once any read has seen a new value, all later reads see it too. It is a *recency* guarantee on individual objects. It differs from *serializability*, which is about isolating transactions; having both is called strict serializability.
- **Where linearizability is genuinely needed**:
  - *Locks and leader election*: everyone must agree on who holds it.
  - *Uniqueness and hard limits*: unique usernames, a seat sold once, a balance that can't go negative, stock that can't be oversold. The decision needs the single latest value.
  - *Cross-channel timing dependencies*: two systems are written in sequence, and a consumer triggered through one reads from the other. For example, a file is uploaded to storage and a job is queued to process it; the worker reads a lagging replica and gets the old file.
- **Which replication approaches can provide it**: single-leader, if reads go to the leader (or synchronous followers) and the leader really is the leader; consensus-based systems, yes; multi-leader, no; leaderless quorums, generally not.
- **The cost, with CAP read narrowly**: during a network partition, a replica that can't reach the others must either refuse requests (staying linearizable) or keep serving them (staying available but not linearizable). CAP covers only that one consistency model and one kind of fault, so it is more useful to reason about each specific operation. Even without faults linearizability is slow, because response time tracks the uncertainty of network delays, which is why many systems give it up for speed.
- **Causality** is the ordering users actually notice: a reply comes after the message it answers, and a read that informed a write comes before that write. Causal consistency needs no global coordination and stays available during network problems; it is the strongest model known to keep both properties. Track it with version vectors, or by passing the last-seen log position along with requests.
- **Lamport timestamps** (a counter plus a node ID) give a total order consistent with causality, but only in hindsight. They can't decide *right now* whether a username claim is the first, because another node might hold a concurrent, lower-numbered claim not yet seen.
- **Total order broadcast** (every node delivers the same messages in the same order) is what a replicated log provides. It is equivalent to consensus and underlies state machine replication. Linearizable operations such as "claim this username" can be built on it: append the claim to the log, read the log back, and the first claim for that name wins.
- **Consensus** means several nodes agreeing on a decision (who is leader, whether a transaction commits, who got the lock, which claim won) such that the decision is final even when some nodes fail. Many problems reduce to it: linearizable compare-and-set, atomic commit, total order broadcast, locks and leases, membership, uniqueness constraints. A single-leader database handles these by making the leader the decision-maker, but choosing a new leader after a failure needs consensus again, or a human, or waiting for the old leader to return.
- **Two-phase commit (2PC)** provides atomic commit across nodes, but if the coordinator crashes after the participants have prepared, they stay stuck holding locks until it recovers. Distributed transactions across different technologies (XA) add operational pain: orphaned in-doubt transactions, the coordinator as a single point of failure, and a failure in one system spreading into the others.
- **Fault-tolerant consensus algorithms** (Raft, Paxos, Zab) need a strict majority of nodes to be reachable, usually assume fixed membership, are sensitive to network timing (leader elections can thrash on flaky networks), and pay a round of voting for every decision.
- **Coordination services** (ZooKeeper, etcd) package consensus as a service: linearizable compare-and-set, total ordering, failure detection through sessions, and change notifications. Use them for leader election, partition assignment, locks, and service membership: small, slowly changing coordination data, not application data.
- Not every system needs consensus. Multi-leader and leaderless systems deliberately avoid it and live with conflicts and merging.

## Questions to ask

- Which specific operations need the single latest value at decision time (uniqueness, quotas, inventory, balances, seat booking, locks, leader election)?
- Are there cross-channel dependencies, where one system is written and a consumer then reads another?
- Must linearizability hold across regions, or only within a region or per key?
- During a network partition, should each operation fail (stay consistent) or proceed and reconcile later (stay available)?
- Is an atomic commit across several different systems needed? Could it be restructured as one log plus idempotent consumers?

## Decision guide

- **Decide per operation.** Usually only a handful of operations need linearizability; most reads can be causal or eventually consistent. Writing that list down is the point.
- **For linearizable operations**: a single-leader database with those reads and writes on the leader (and fencing on failover), or a store replicated by consensus. Check vendor consistency claims against independent testing where possible.
- **Uniqueness**: a unique constraint in one database. At larger scale, route all claims for the same key to one log partition and let a single consumer decide in log order (see `correctness-and-operations.md`).
- **Leader election, locks, membership**: a coordination service plus fencing tokens. Never invent a consensus algorithm.
- **Across systems**: avoid 2PC/XA on hot paths. Prefer one durable log as the place where the decision is made, idempotent downstream consumers, and compensating actions for business-level reversals.
- **Cheaper alternative where it suffices**: causal consistency, using session tokens or version vectors, so users see their own writes and related updates in order.
- **Cross-channel dependencies**: have the consumer read the exact version the message refers to (include a version number or content hash in the message), or route both steps through one ordered channel.

Examples to verify against current documentation: etcd and ZooKeeper for coordination; Spanner, CockroachDB, YugabyteDB, and FoundationDB as databases offering strongly consistent operations across nodes; Raft libraries for embedding consensus in a service.

## Red flags

- Uniqueness, inventory, or balance limits enforced by check-then-insert against an eventually consistent store or an asynchronous replica.
- Home-grown leader election, locking, or consensus.
- XA/2PC across heterogeneous systems on the request path.
- "Strong consistency" demanded everywhere (needless cost), or nowhere (hidden correctness bugs).
- A pipeline that writes to store A and notifies through channel B, whose consumer reads a replica that may lag.
- Automatic failover built without a consensus mechanism.

## What to record

- A per-operation consistency table: operation, required guarantee (linearizable, causal, eventual), mechanism.
- The behavior of each operation during a network partition (fail, or proceed and reconcile).
- Coordination needs and which service provides them.
