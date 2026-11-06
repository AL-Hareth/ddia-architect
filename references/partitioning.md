# Partitioning (sharding)

Book: Ch 6, "Partitioning" (sections: Partitioning of Key-Value Data; Partitioning and Secondary Indexes; Rebalancing Partitions; Request Routing)

## Core ideas

- Partitioning spreads data and load across machines once a single node can no longer hold the data or absorb the writes. It is usually combined with replication: each partition is itself replicated.
- The goal is an even spread. Skew creates **hot spots**, where one partition does most of the work and the rest of the cluster sits idle.
- **Key-range partitioning**: each partition owns a contiguous range of keys. Range scans are efficient, but sequential keys (timestamps, auto-increment IDs) send every new write to the same partition. The fix is to lead the key with something else (sensor ID before timestamp), at the cost of range queries across that leading part.
- **Hash partitioning**: each partition owns a range of hash values. Load spreads evenly, but efficient range queries on the key are lost. A *compound key* gets both: hash the first part to choose the partition, keep the rest sorted inside it (all of one user's events, ordered by time).
- **Keys that are hot by nature** (a celebrity, a viral post, a giant tenant) defeat even hashing. An application-level fix is to split such a key by appending a small random suffix, spreading its writes over several partitions, and then read and merge all the pieces. That is worth it only for known-hot keys, and it requires tracking which keys are split.
- **Secondary indexes on partitioned data**:
  - *Local (document-partitioned)*: each partition indexes only its own data. A write touches one partition, but a query on the secondary attribute must ask every partition and wait for the slowest (scatter/gather), which amplifies tail latency.
  - *Global (term-partitioned)*: the index is partitioned by the indexed value. A read hits one index partition, but a write touches several, and the index is usually updated asynchronously, so it lags behind.
- **Rebalancing**:
  - Never assign data to nodes with hash mod N: changing N moves almost every key.
  - *A fixed number of partitions*, many more than there are nodes, moved whole between nodes. Simple, but the count is fixed up front: too few and partitions grow huge; too many and per-partition overhead adds up.
  - *Dynamic partitioning*: split partitions that grow too large, merge ones that shrink (an empty database starts with a single partition unless pre-split).
  - *Partitions proportional to nodes*: each node holds a fixed number of partitions.
  - Fully automatic rebalancing combined with automatic failure detection can cascade: an overloaded node looks dead, its load moves to other nodes, which then look dead too. Keep a human in the loop when rebalancing large clusters.
- **Request routing**: any node forwards requests it doesn't own, or a routing tier sits in front, or clients are partition-aware. The mapping of partitions to nodes is usually held in a coordination service (ZooKeeper, etcd) or spread by gossip.

## Questions to ask

- Does the three-year data size or write rate actually exceed one node with headroom? If not, why partition now?
- Which key do most queries and transactions include (tenant, user, account, device)? That is the candidate partition key.
- Is activity heavily skewed across keys (power-law users, one giant customer, one hot product)?
- Are range queries on the key needed (time ranges, alphabetical listings)?
- Which queries filter on attributes other than the partition key, how often, and how latency-sensitive are they?
- Will any operation need to update several partitions atomically?

## Decision guide

- **Don't partition prematurely.** If one node suffices, record the trigger for partitioning instead, but pick IDs and a data model that would allow partitioning by tenant or user later.
- **Partition key** = the unit that most queries and transactions are scoped to. Cross-partition transactions are expensive or unavailable, so design so that the important invariants live within a single partition.
- **Hash** for an even spread; **range** only when range scans on the key are essential (then defend against sequential-key hot spots); **compound** keys for "everything belonging to X, sorted by time".
- **Plan for hot keys explicitly**: identify them, then use key splitting, a dedicated partition, caching, or an admission queue.
- **Secondary queries**: local indexes if writes dominate and secondary queries are rare or analytical; a global index if secondary reads are on the hot path and some lag is acceptable; or a separate derived store (such as a search index) fed by change data capture.
- **Rebalancing**: many fixed partitions or dynamic splitting, with operator-approved moves for large clusters.
- Prefer an established or managed system that handles routing and rebalancing over a hand-rolled sharding layer, and understand how it does it.

Examples to verify against current documentation: Citus, Vitess, CockroachDB, YugabyteDB, or Spanner for partitioned SQL; Cassandra/ScyllaDB or DynamoDB for partitioned key-value and wide-column data.

## Red flags

- Assigning data to nodes with hash(key) mod N.
- A monotonic key (timestamp, sequence) leading a range-partitioned key on a write-heavy table.
- Scatter/gather queries on a latency-critical path.
- Invariants that span partitions with no enforcement plan (see `transactions.md` and `correctness-and-operations.md`).
- Partitioning a dataset that fits comfortably on one machine.
- A fixed partition count chosen without considering three to five years of growth.

## What to record

- Whether to partition now, and the trigger if not.
- The partition key, the scheme (hash, range, compound), and how hot keys are handled.
- The secondary-index strategy and the acceptable lag.
- The rebalancing approach and who approves moves.
