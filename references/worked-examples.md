# Worked examples

Two condensed sessions showing how depth, pacing, and phrasing change with the stakes. They are illustrations of the approach, not scripts to copy.

## Example 1: Lite (a habit-tracking API)

**Developer:** I want to build a habit tracker backend in Node. Users create habits and check them off each day. Can you set up the project?

**Agent, turn 1:**
> Before I scaffold anything, a quick plan. This looks like a Lite one: modest data and ordinary invariants. Three questions, each with a default you can just accept:
> 1. Users in the first year? *(default: under 10k)*
> 2. Do the mobile apps need to work offline and sync later? *(default: no, online only)*
> 3. Anything social later, like friends or shared streaks? *(default: maybe next year)*

**Developer:** Defaults are fine. Friends probably next year.

**Agent, turn 2** (requirements and plan in a single message):
> - **Numbers**: ~10k users × ~5 check-ins a day ≈ 50k writes/day ≈ 0.6/s average, maybe 10/s at peak. Well under 1 GB a year. Everything fits on one small database node for many years. `ASSUMPTION`: online-only clients.
> - **Invariant**: one check-in per habit per local calendar day, and client retries must not double-count.
> - **Data model**: relational (users, habits, check-ins). Friendships later are many-to-many, which relational handles naturally; a document store would get awkward once social features arrive.
> - **Store**: one managed PostgreSQL instance with automated backups and point-in-time recovery. No cache, queue, or search engine: nothing in the requirements needs them, and each would add operations work and another copy of the data to keep consistent. Revisit a cache if p99 for loading the habit list exceeds 200 ms.
> - **Correctness**: a unique constraint on (habit_id, local_date) makes duplicate check-ins impossible and client retries harmless. Compute the local date on the server from the user's stored time zone, not from the server clock and not blindly from the client.
> - **Evolution**: migrations through a migration tool, additive changes only, so old and new app versions can run side by side.
> - **Red-flag review**: nothing critical. Time-zone handling was the one subtle risk, covered above.
>
> Does this look right? If so, I'll scaffold the project with this schema and the migration setup.

*What this shows*: restraint (one store, no speculative components), numbers even at small scale, the one real invariant identified and enforced declaratively, and the whole plan in one message.

## Example 2: Deep (ticket sales with flash on-sales)

**Developer:** We're building ticketing for concerts. Reserved seating, payments through a provider, and big artists sell out in minutes. Design the backend.

**Agent, turn 1:**
> This calls for a Deep plan: money, a hard invariant (a seat is never sold twice), and extreme peaks. First round:
> 1. The worst on-sale you expect: how many people in the first minute, for how many seats? *(default: 200k people for 20k seats)*
> 2. Do customers hold seats while they check out? For how long? *(default: 10-minute hold)*
> 3. One region or several? *(default: one region)*
> 4. May an acknowledged sale ever be lost in a crash? *(default: never)*

**Developer:** 300k people for 25k seats, 8-minute holds, one region, never lose a sale.

**Agent, turn 2 (Gate 1):**
> Let me confirm the foundations:
> - Browsing and search are read-heavy and may be a few seconds stale. The seat map should be near-fresh. Holding and buying must be strictly correct.
> - Peak: ~300k users in the first minute, nearly all on one event, so a single hot key.
> - RPO = 0 for orders. Payment-provider calls can time out with an unknown outcome.
>
> Correct?

**Developer:** Correct.

**Agent, turns 3–5** (key decisions, each with options and trade-offs):
> - **System of record**: a relational database, single leader, with a synchronous replica so no acknowledged order is lost. We accept a few milliseconds of extra write latency.
> - **Seat holds**: one conditional UPDATE that claims all requested seats only if each is free or its hold has expired, rolled back unless the affected row count matches the request. Hold expiry is evaluated with the database's own clock, so there is one clock and no cross-machine comparison. *Rejected*: a lock in a cache with a TTL. It has no fencing token, so an app server that paused could still sell a seat after its lock expired.
> - **The hot event**: partitioning can't help, because one event's seats form a single hot key. Instead, an admission queue (a virtual waiting room) caps concurrent checkouts at a rate the database is proven to sustain, and everyone else waits in line politely. This is the riskiest assumption, so the first slice is a spike that load-tests the hold path on one event's seats.
> - **Payments**: an idempotency key created when checkout starts, stored with the order under a unique constraint and passed to the provider. After a timeout, look the payment up by that key instead of charging again. Orders move through explicit states: held, paying, paid, or released.
> - **Derived data**: event search and the seat-map availability view are fed from the database by CDC; seconds of staleness is acceptable there, and purchases always go to the system of record. *Caught in the red-flag review*: the initial idea of having the application write to the search index directly (a dual write), replaced with CDC.
> - **Emails and e-tickets**: an outbox row written in the same transaction as the order, published to a queue; consumers deduplicate on order ID.

**Agent, turn 6** (plan and approval):
> I've written `docs/ARCHITECTURE.md` with the ADRs, a failure-mode table (primary failover, payment-provider timeout, queue backlog, CDC lag, waiting-room overload), and slices ordered by risk: (1) hold-path load-test spike, (2) holds and purchase with idempotency, (3) waiting room, (4) CDC to search and the seat map, (5) outbox notifications. Approve, or should anything change first?

*What this shows*: invariants drive the design; the hot key is recognized as something partitioning can't fix; every rejected alternative has a stated reason; the riskiest assumption is tested before anything is built on it.
