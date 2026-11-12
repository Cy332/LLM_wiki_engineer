---
title: "PostgreSQL + Outbox Pattern Revamped — Part 1"
author: "Mohammad S. Dousti (msdousti)"
source_url: "https://dev.to/msdousti/postgresql-outbox-pattern-revamped-part-1-3lai"
published_at: 2025-06-08
ingested_at: 2026-11-12
---

# PostgreSQL + Outbox Pattern Revamped — Part 1

> Source: [DEV Community - Mohammad S. Dousti](https://dev.to/msdousti/postgresql-outbox-pattern-revamped-part-1-3lai)
> Published: 2025-06-08

The outbox pattern is a well-known design pattern used in distributed systems to ensure messages are sent reliably, by first storing messages in a local database before sending them to a message broker. The `outbox` table often contains fields like `id`, `payload`, `created_at`, and `published_at`. There's a scheduled process that picks up N messages from the outbox, sends them to the message broker, and marks them as published.

While this seems pretty straightforward, I've seen quite a number of incidents corresponding to the outbox pattern. In this blog post, I'll go over some of these incidents and discuss how to improve your setup to avoid them. With the last example, I'll demonstrate an issue that is often overlooked, and suggest a "revamped" solution that can significantly improve performance and reliability. This solution is based on partitioning the outbox table.

> Note: Some people may favor Debezium (CDC via PostgreSQL WAL) for the outbox pattern. However, teams often implement polling-based outbox manually to:
> 1. Avoid the operational complexity of Debezium / Kafka Connect.
> 2. Publish to brokers other than Kafka.
> 3. Avoid logical replication slot issues in PostgreSQL (which can block WAL cleanup during outages or complicate major version upgrades).

## Incident 1: Sorting by created_at

In distributed systems, getting a total ordering of events based on local timestamps is a fatal error. When each pod of the microservice labels the outbox messages with its own `created_at` timestamp and the publisher sorts by `created_at`:

```sql
SELECT * FROM outbox
WHERE published_at IS NULL
ORDER BY created_at
LIMIT 10;
```

Because pods have slightly different clocks (clock drift of even fractions of a millisecond), messages are sent out of order in high-concurrency systems. Instead, use a database-supplied monotonic sequence number (`GENERATED ALWAYS AS IDENTITY`):

```sql
SELECT * FROM outbox
WHERE published_at IS NULL
ORDER BY id
LIMIT 10;
```

## Incident 2: Using the wrong data types

1. **Primary Key / Sequence Exhaustion**: Always use `INT8` (`bigint`) for `id`, never `INT4` (`integer`, which caps at ~2.1 billion and will cause an outage in high-throughput event tables).
2. **Timestamps**: Always use `TIMESTAMPTZ` (timezone-aware).
3. **Payload (`JSON` vs `JSONB`)**: `jsonb` is great if you need to index inside the payload, but it is slower to parse/store and **rejects `\x00` (Unicode NULL character)** with `ERROR: invalid input syntax for type json`. For pure message relay, `JSON` or `TEXT`/`BYTEA` is safer and faster.

```sql
CREATE TABLE outbox (
    id INT8 GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    payload JSON NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at TIMESTAMPTZ
);
```

## Incident 3: Incorrect indexing

For the polling query (`WHERE published_at IS NULL ORDER BY id LIMIT 10`), the optimal index is a **Partial Index** on `(id)`:

```sql
CREATE INDEX idx1 ON outbox (id)
WHERE published_at IS NULL;
```

- **Bad Alternative 1 (No Partial Index)**: Relying on `outbox_pkey` scans all historical published rows and filters `published_at IS NULL`, degrading over time. Sorting by unindexed `created_at` forces a `Seq Scan` + `Sort`.
- **Bad Alternative 2 (Indexing `published_at` instead of `id`)**:
  ```sql
  -- ❌ Don't do this ❌
  CREATE INDEX idx2 ON outbox (published_at) WHERE published_at IS NULL;
  ```
  This builds a B-Tree full of `NULL` values and cannot help with `ORDER BY id`.

## Incident 4: Picking too many messages at each iteration

PostgreSQL hates long-running transactions. The polling cycle opens a transaction, fetches N messages, performs network I/O to publish to the message broker, and commits.
- If N is too large (e.g., > 1,000), network latency stretches the DB transaction duration, blocking `autovacuum` and holding locks.
- Any failure rolls back the entire batch of N messages, causing duplicate reprocessing.
- **Best Practice**: Pick ~100 messages per iteration (never > 1,000). Batch-publish to the broker, grouping by partition key (e.g., `user_id` or `account_id`) so different groups can be sent in parallel while preserving per-entity ordering.

## Incident 5: Poison pill messages

A message that permanently fails broker publication (e.g., incompatible schema in Schema Registry) is a "poison pill". If unhandled, it causes the batch transaction to roll back and retry infinitely, halting the outbox queue.
- **Mitigation**: Group messages and catch exceptions per group so healthy groups still commit, and trigger immediate alerting on poison pills (or mark retry count / move to Dead Letter Queue).

## Incident 6: Inefficient index and visibility checking (The Heap Fetch Disaster)

In high-throughput systems where messages are rapidly inserted and marked `published_at = NOW()`, the partial index `idx1` accumulates dead tuples before `autovacuum` can clean them up and update the Visibility Map (VM).

When running:
```sql
EXPLAIN (ANALYZE)
SELECT id FROM outbox WHERE published_at IS NULL ORDER BY id LIMIT 1000;
```
Instead of `Heap Fetches: 0` (0.13 ms), concurrent load causes PostgreSQL's `Index Only Scan` to perform **up to 96,633,220 Heap Fetches**, spiking execution time to **18,553 ms (18.5 seconds — a 5 orders of magnitude slowdown)**! Furthermore, starting in PostgreSQL 14, `VACUUM` may skip index cleanup (`INDEX_CLEANUP AUTO`) when removable entries are below a threshold.

## How to Fix the Issue: Partitioned Outbox Table

Partition the `outbox` table using `PARTITION BY LIST (published_at)`:

```sql
CREATE TABLE outbox (
    id INT8 GENERATED ALWAYS AS IDENTITY,
    payload JSON NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at TIMESTAMPTZ
)
PARTITION BY LIST (published_at);

-- Partition 1: Unpublished messages (published_at IS NULL)
CREATE TABLE outbox_unpublished 
    PARTITION OF outbox
    FOR VALUES IN (NULL);

-- Partition 2: Published messages (published_at IS NOT NULL)
CREATE TABLE outbox_published 
    PARTITION OF outbox
    DEFAULT;

-- Index ONLY the unpublished partition
CREATE INDEX idx1 ON outbox_unpublished (id);
```

### Crucial Gotcha When Updating a Partitioned Outbox Table

Never run an `UPDATE` without the partition key in the `WHERE` clause:
```sql
-- ❌ WRONG: Causes Seq Scan on BOTH outbox_unpublished and outbox_published!
UPDATE outbox SET published_at = NOW() WHERE id = 1;

-- ✅ CORRECT: Enables Partition Pruning, scanning ONLY outbox_unpublished via idx1
UPDATE outbox
SET published_at = NOW()
WHERE id IN (SELECT id FROM t)
  AND published_at IS NULL;
```

When `published_at` changes from `NULL` to `NOW()`, PostgreSQL automatically deletes the row from `outbox_unpublished` and inserts it into `outbox_published`.
- **Result**: Under heavy parallel insert + publish load, query execution time stays rock-solid at **1–3 ms** (only ~1,000 heap fetches).
- **Instant Cleanup**: Old published messages in `outbox_published` can be purged instantaneously via `TRUNCATE outbox_published` without generating dead tuples or index bloat!
