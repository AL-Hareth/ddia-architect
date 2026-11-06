# Networks, clocks, and process pauses

Book: Ch 8, "The Trouble with Distributed Systems" (sections: Unreliable Networks; Unreliable Clocks; Process pauses; Knowledge, Truth and Lies)

## Core ideas

- **Partial failure** is the defining property of distributed systems: some parts work and some don't, and it is often impossible to tell which. The job is to build a reliable whole from unreliable parts. If the problem fits on one machine, staying on one machine avoids this whole class of problems.
- **Networks**: a request may be lost, queued, or delayed; the remote node may have crashed, paused, or be overloaded; the reply may be lost or delayed. From the sender's side all of these look the same: no response.
  - A **timeout** is the only practical detector, and it says nothing about whether the operation happened.
  - Too short, and healthy-but-slow nodes are declared dead, their work moves elsewhere, and the extra load can topple other nodes (a cascading failure). Too long, and recovery is slow. Base timeouts on the measured distribution of response times; adaptive failure detectors adjust automatically.
  - Most network delay is *queueing* (in switches, OS buffers, behind noisy neighbours on shared hardware, in TCP retransmission), which is why delays on shared infrastructure have no upper bound.
  - Network partitions happen in real datacenters. Decide what each component does when it can't reach another.
  - A "limping" node, degraded but not dead, can be harder to handle than one that crashed cleanly.
- **Clocks**:
  - *Time-of-day* clocks can jump backwards or forwards when synchronized and drift between synchronizations; synchronization accuracy is limited by network delay. *Monotonic* clocks only move forward and are right for measuring elapsed time on one machine, but their values mean nothing across machines.
  - Ordering events across nodes by wall-clock timestamp is unsafe. A node with a slightly fast clock makes its writes "win", silently discarding writes from nodes whose clocks run slow (last-write-wins). Use logical clocks or a single sequencer for ordering.
  - Treat any clock reading as an interval with uncertainty. Systems that use timestamps for global ordering bound that uncertainty explicitly and wait it out (the approach Spanner takes).
- **Process pauses**: a thread can stop for a long time at any point, due to garbage collection, VM suspension or live migration, OS scheduling, swapping, or slow synchronous disk I/O. A process that checked "I still hold the lease" and then paused may resume and act after the lease expired, without realizing anything happened.
- **Truth is decided by a majority**: a node can't trust its own belief that it is alive, the leader, or the lock holder. Such decisions go through a quorum.
- **Fencing tokens**: the lock or lease service hands out a number that increases with every grant, and the protected resource rejects any request carrying a lower number than one it has already seen. That makes a paused former lock holder harmless. A lock without fencing is advisory at best.
- **Byzantine faults** (nodes that lie) usually don't need tolerating inside one organization's datacenter, but validate inputs, use checksums, and treat external clients and peers as untrusted.
- **System models**: realistic designs assume *partially synchronous* timing (usually well-behaved, occasionally wild) and *crash-recovery* nodes (they can crash and come back with only what was on disk). Distinguish **safety** properties, which must never be violated (uniqueness, no data loss), from **liveness** properties, which say something good eventually happens (a request eventually gets a response).

## Questions to ask

- For each network call in the design: what is the timeout, the retry policy, the idempotency mechanism, and the behavior when the outcome is unknown?
- Does anything rely on "only one node does X at a time": a leader, a scheduler, a lock, a lease? How is a stale holder prevented from acting?
- Does any logic compare timestamps from different machines (ordering, expiry, conflict resolution)?
- What happens on a network partition between each pair of components or regions?
- What runtime pauses are realistic for this stack (garbage-collected languages, VMs, containers)?

## Decision guide

- **Every remote call**: an explicit timeout based on measured latency, bounded retries with exponential backoff and jitter, an idempotency key for anything non-idempotent, and a circuit breaker or load shedding so one slow dependency can't exhaust its callers.
- **Unknown outcome after a timeout**: look up the result by idempotency key, or retry safely. Never assume the operation failed.
- **Durations and timeouts**: monotonic clocks. **Ordering across nodes**: logical clocks, version numbers, or sequence numbers from a single leader or log, not wall-clock time.
- **Mutual exclusion**: use leases from a consensus-backed coordination service (etcd, ZooKeeper) or the database's own locking, and pass a fencing token that the resource checks. If the resource can't check tokens, make the operations idempotent or conflict-tolerant instead of relying on the lock.
- **Failure detection**: tune to avoid flapping; consider human approval for rare, high-impact actions such as promoting a new primary.
- **Test it**: inject faults on the critical paths (kill processes, drop and delay packets, skew clocks, pause processes) before production finds the bugs.

## Red flags

- Wall-clock timestamps used to order events or resolve conflicts across machines.
- Durations measured with the time-of-day clock.
- A distributed lock (for example a cache key with a TTL) protecting a resource with no fencing token.
- A node acting on "I am the leader" without that being re-validated by a quorum or fenced at the resource.
- A timeout treated as a definite failure, with compensation applied as if nothing happened.
- Unbounded or synchronized retries with no backoff or jitter (retry storms).
- No defined behavior for a network partition between the application and its database or broker.

## What to record

- The timeout, retry, and idempotency policy for each dependency.
- Every single-holder role (leader, lock, scheduler) and how it is fenced.
- Where wall-clock time is used and why it is safe there.
- The planned fault-injection tests.
