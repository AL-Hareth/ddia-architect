# Storage engines, indexes, OLTP vs analytics

Book: Ch 3, "Storage and Retrieval" (sections: SSTables and LSM-trees; B-trees; Other indexing structures; Keeping everything in memory; Transaction Processing or Analytics?; Column-oriented storage)

## Core ideas

- **Every index is a trade-off**: it speeds up some reads and slows down every write. Derive indexes from the actual query list, not speculatively.
- **Two families of OLTP storage engine**:
  - *Log-structured* (LSM-trees: an in-memory buffer flushed to sorted immutable files, merged by background compaction, with Bloom filters to skip files that can't contain a key). Random writes become sequential, which typically means higher write throughput and better compression. Costs: compaction competes with foreground requests and can raise tail latency; under sustained heavy writes it can fall behind until disk fills; a key may exist in several files, so a read may check several places.
  - *B-trees* (fixed-size pages updated in place, plus a write-ahead log for crash safety). The default in relational databases. Each key lives in exactly one place, which makes reads predictable and suits transactional locking. Costs: each write touches a page and the log (write amplification), and pages fragment.
  - Rule of thumb: LSM engines tend to win on writes and B-trees on reads, but the only trustworthy answer is a benchmark with this workload.
- **Index variants**: clustered (the row stored inside the index), covering (extra columns in the index so it can answer a query alone), composite (several columns; order matters), multi-dimensional (R-trees or geohashes for location), full-text and fuzzy (inverted indexes, edit-distance matching, as in Lucene).
- **In-memory stores** are fast mostly because they skip encoding data into disk-friendly structures, not only because they avoid disk reads. They can offer richer data structures (sets, queues, sorted sets). Durability needs explicit configuration (append logs, snapshots, replication), or the store must be treated as a disposable cache.
- **OLTP and OLAP are different workloads**:

| | OLTP (operational) | OLAP (analytical) |
|---|---|---|
| Used by | End users, through the application | Analysts, BI, data science |
| Reads | A few records, by key | Aggregates over millions of rows |
| Writes | Small, random, low latency | Bulk loads or event streams |
| Bottleneck | Seeks and latency | Scan bandwidth |
| Fits | Row-oriented and indexed | Column-oriented and compressed |

- **Data warehouses** keep heavy analysis off the OLTP database. A typical layout is a large *fact* table of events surrounded by *dimension* tables (a star schema; a snowflake schema when the dimensions are normalized further).
- **Column-oriented storage** keeps each column separately, so a query reads only the columns it needs. Columns compress extremely well (bitmaps, run-length encoding) and can be processed fast. Writes are batched (buffered, then merged in). Precomputed aggregates and data cubes trade query flexibility for speed.

## Questions to ask

- What is the write pattern: append-only events, in-place updates, bulk loads? The peak write rate?
- What is the read pattern: point lookups, range scans, geospatial, full-text, aggregates?
- What analytics or reporting is needed, by whom, how fresh, and how heavy?
- Does the hot working set fit in memory?
- Which query needs which index? Is any index serving no listed query?

## Decision guide

- **Default OLTP**: a B-tree-based relational database.
- **Very high write throughput** (events, metrics, logs, time series, IoT): consider an LSM-based or time-series store. Budget disk and I/O headroom for compaction, and watch tail latency.
- **Analytics**:
  - Light, internal reporting: a read replica, so the primary isn't slowed down.
  - Heavy or broad analytics: a columnar warehouse or lakehouse fed by ETL or change data capture from the system of record, with a stated freshness. Small datasets can use an embedded columnar engine.
- **Full-text search**: a search index maintained as *derived* data from the system of record (see `derived-data.md`), not written separately by the application.
- **Geospatial**: a store with spatial indexes (R-tree or geohash).
- **Caches and in-memory stores**: decide explicitly whether each is a disposable cache (can be lost and rebuilt) or a system of record (persistence and replication configured and tested).

Examples to verify against current documentation: PostgreSQL or MySQL for B-tree OLTP; RocksDB-based engines, Cassandra/ScyllaDB, or time-series databases for write-heavy work; ClickHouse, BigQuery, Snowflake, Redshift, or DuckDB for columnar analytics; Elasticsearch/OpenSearch for search; Redis/Valkey for in-memory structures.

## Red flags

- Heavy analytical queries running on the OLTP primary.
- No headroom for compaction, vacuuming, or write amplification on a write-heavy store.
- An engine chosen from someone else's benchmark that doesn't resemble this workload.
- An in-memory store holding the only copy of important data with persistence not configured.
- Indexes added "just in case", or the primary access pattern lacking one.

## What to record

- A store for each workload type (OLTP, analytics, search, cache), with its role and freshness.
- The index list, each index tied to the query it serves.
- How analytical data gets from the system of record to the analytical store, and how stale it may be.
