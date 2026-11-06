# Requirements, load, and performance

Book: Ch 1, "Reliable, Scalable, and Maintainable Applications" (sections: Reliability; Describing load; Describing performance; Approaches for coping with load; Maintainability)

## The three concerns

- **Reliability**: the system keeps working correctly when things go wrong. A *fault* is one component deviating from its spec; a *failure* is the system as a whole letting its users down. Design for tolerance so faults don't escalate into failures, and test that tolerance by deliberately injecting faults.
  - Hardware faults are mostly random and independent: use redundancy, and software that tolerates losing a whole machine (which also makes rolling upgrades possible).
  - Software faults are systematic and correlated: a bad input that crashes every node at once, a runaway process, a dependency that slows down, cascading failures. Challenge assumptions, test thoroughly, isolate processes, let them crash and restart, monitor, and have the system check its own invariants while it runs.
  - Human error, especially configuration changes, is a leading cause of outages: interfaces that make the right thing easy, sandboxes to experiment safely, automated tests, fast rollback, gradual rollouts, tools to recompute data, and detailed telemetry.
- **Scalability**: having a strategy for keeping performance acceptable as load grows. The word only means something once load and performance are described with numbers.
- **Maintainability**: operability (easy to keep running), simplicity (no accidental complexity, good abstractions), evolvability (easy to change). Most of a system's lifetime cost is operating and changing it, not writing it.

## Question bank

Pick the questions that matter for this system; don't ask all of them.

**Users and data**
- Who are the users (people, devices, other services)? How many now, in 12 months, in 3 years?
- What are the core entities? Rough size of a record? How many records, growing how fast?
- How long must data be kept? Is some of it append-only by nature (events, logs, sensor readings)?

**Operations**
- The top operations by frequency, and by business importance.
- For each: read or write? By key, by range, by text search, by aggregate?
- Read/write ratio, overall and for the hottest operation.
- Fan-out: does one write cause many reads or updates elsewhere (feeds, notifications, denormalized copies)? Does one read touch many records or services?
- Is activity spread evenly across keys, or do a few keys (celebrities, big tenants, flash-sale items) get much of it?

**Targets**
- Latency target per key operation, as a percentile, measured where (ideally at the client)?
- Throughput target and deadline for batch work ("the nightly job finishes by 6 am").
- Freshness: how stale may each derived view be (search results, dashboards, recommendations)?

**Reliability**
- Worst acceptable outcome of a crash: lose the last few seconds of writes, or nothing at all?
- Acceptable downtime per incident and per month?
- Which failures must the system survive: a process, a machine, an availability zone, a region?

**Team and operations**
- Who operates this, with what experience? Is there an on-call rotation?
- Are managed services allowed? Any budget ceiling?

## Back-of-envelope estimation

Do it together with the developer and write the arithmetic into the plan so anyone can check it.

1. **Rate**: users × actions per user per day ÷ 86,400 ≈ average operations per second.
2. **Peak**: average × peak factor. Daily cycles commonly give 2–10×. Launches, flash sales, scheduled jobs, and push-notification herds can give 100× or more for a few minutes. Ask what the worst minute looks like.
3. **Storage**: records per day × bytes per record × retention, then × an overhead factor for indexes and storage-engine overhead (often 1.5–3×), then × the replication factor.
4. **Fan-out**: for each write, how many derived updates or deliveries does it trigger? For each read, how many records or services does it touch?
5. **Horizon and headroom**: design for the 12–36 month figure, not today's, with 2–3× headroom on top.

Rough reference magnitudes, for sanity checks only (benchmark before relying on them): a round trip inside one datacenter is about half a millisecond, a cross-continent round trip is on the order of 100 ms, and an SSD random read is around 0.1 ms. A single well-tuned relational database server on current hardware commonly handles thousands to tens of thousands of simple transactions per second and several terabytes of data, depending heavily on the workload.

## Stating performance targets

- **Percentiles, not averages.** The median describes the typical request; p95, p99, and p999 describe the tail. A mean hides outliers.
- **The tail matters disproportionately.** The slowest requests often come from the heaviest, most valuable users. When one user request fans out to many backend calls, the slowest call sets the response time, so even rare slowness in a backend shows up often at the top.
- **Measure at the client.** Queueing delay and head-of-line blocking (a few slow requests holding up the fast ones queued behind them) are invisible from the server's own timing.
- **Load-test with an open loop.** The load generator must send requests at a fixed rate regardless of how fast responses arrive. One that waits for each response before sending the next keeps queues artificially short and flatters the results.
- **Aggregate correctly.** To combine percentiles across machines or time windows, merge histograms (HdrHistogram, t-digest and similar). Never average percentiles.
- Good target: "p99 of *add to cart* under 200 ms at 3k req/s, measured at the API gateway." Bad target: "fast".

## Do we need a distributed system at all?

There are three legitimate reasons to distribute (scale beyond one machine, fault tolerance, low latency for users spread across the world) and a large, permanent cost in complexity. Decide per pressure, not wholesale:

| Pressure | First response | Only later, if still needed |
|---|---|---|
| Read throughput | Indexes, query tuning, caching, read replicas | Partitioning |
| Write throughput or data size | Bigger machine, batching, a write-efficient storage engine | Partitioning (`partitioning.md`) |
| Analytics slowing down OLTP | A read replica for reporting | A separate analytical store |
| Availability | A replica with tested failover, plus backups | Multi-zone, then multi-region |
| Global latency | CDN and caches, read replicas near users | Multi-region writes |

If the three-year estimate fits on one node with headroom, say so and design for one node plus replicas, and write down the trigger that would require partitioning. An architecture built for one scale rarely fits a scale ten times larger anyway, and early on the ability to change quickly is worth more than capacity for hypothetical load. Still, avoid choices that make later scaling needlessly hard, such as ID schemes or data models that prevent splitting data by tenant or user.

## Red flags

- Targets stated as averages, or no numbers at all.
- Designing for enormous scale with no load estimate, or for today's load when 10× growth is expected within a year.
- No answer to "who operates this at 3 am?"
- Load tests that wait for each response before sending the next request.

## What to record

- The load parameters with numbers, the arithmetic behind them, and which ones are assumptions.
- Latency and throughput targets as percentiles per key operation.
- RPO, RTO, and the failure scope the system must survive.
- The triggers at which the design must change ("partition when the main table passes ~2 TB or writes pass ~5k/s").
