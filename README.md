# pg-system-design-playbook

> Production-grade techniques for optimizing PostgreSQL, caching layers, and distributed backend systems.

---

## Overview

A hands-on reference for backend engineers covering real-world patterns in database optimization, indexing, caching, sharding, transaction management, and security — built from production experience, not theory.

Practices align with **OWASP Top 10**, **NIST guidelines**, and **PCI DSS** (for payment systems).

---

## Table of Contents

1. [Network & Latency Optimization](#1-network--latency-optimization)
2. [PostgreSQL Storage Optimization](#2-postgresql-storage-optimization)
3. [Indexing Strategies](#3-indexing-strategies)
4. [Query Optimization](#4-query-optimization)
5. [Caching Strategies](#5-caching-strategies)
6. [Partitioning](#6-partitioning)
7. [Sharding](#7-sharding)
8. [Transactions & Consistency](#8-transactions--consistency)
9. [Security Best Practices](#9-security-best-practices)

---

## 1. Network & Latency Optimization

In many systems, network latency dominates query execution time — not the query itself.

**Scenario:** Node.js backend on AWS EKS, PostgreSQL in a different availability zone.

| Metric | Value |
|--------|-------|
| Query execution time | < 1 ms |
| Total round-trip time | 12–15 ms |

**Root cause:** Multiple hops — Pod → Service Mesh → Load Balancer → AZ → Database.

**Fix:** Co-locate services in the same AZ, or use Unix domain sockets for local connections.

```js
import { Pool } from "pg";

const pool = new Pool({
  host: "/var/run/postgresql",
  database: "app",
});
```

**Result:** Latency drops to ~1.5 ms, throughput increases from ~100k to ~215k TPS with no infrastructure changes.

**Security:** Set socket permissions to `700`, apply OS-level least privilege, use TLS when switching to TCP (OWASP A02).

---

## 2. PostgreSQL Storage Optimization

### Column Alignment

PostgreSQL pads columns to align with type boundaries. Reordering columns by descending type size reduces padding and improves storage efficiency.

```sql
-- Inefficient layout
CREATE TABLE logs (
  message    TEXT,
  created_at TIMESTAMPTZ,
  status     INT,
  user_id    BIGINT
);

-- Optimized layout (descending size: 8B → 8B → 4B → variable)
CREATE TABLE logs_opt (
  created_at TIMESTAMPTZ,
  user_id    BIGINT,
  status     INT,
  message    TEXT
);
```

**Result:** ~12% storage reduction on a 200M-row table (420 GB → 366 GB), better buffer cache utilization.

### Zero-Cost Column Addition

```sql
ALTER TABLE users
ADD COLUMN is_verified BOOLEAN DEFAULT false;
```

No table rewrite. Instant migration. Safe under high traffic. PostgreSQL stores the default in the catalog, not per-row, until the row is updated.

---

## 3. Indexing Strategies

### Index Types

**Single-Column Index** — for equality lookups on a primary or unique field.

```sql
CREATE INDEX idx_users_email ON users (email);
```

**Composite Index** — for combined filters. Column order matters; the leading column must appear in the query.

```sql
CREATE INDEX idx_posts_user_date ON posts (user_id, created_at);
```

**Unique Index** — enforces data integrity at the storage layer.

```sql
CREATE UNIQUE INDEX idx_users_phone ON users (phone_number);
```

**Full-Text (GIN) Index** — for word-level search across text fields.

```sql
CREATE INDEX idx_articles_content ON articles USING GIN (content_tsv);
```

**Hash Index** — very fast equality lookups, not suitable for range queries.

```sql
CREATE INDEX idx_sessions_token ON sessions USING HASH (token);
```

### Functional Index for Memory Efficiency

A standard index on a high-cardinality text column (e.g., 70M emails) can reach 1.1 GB and spill to disk during peak hours. A hash functional index reduces this significantly.

```sql
CREATE INDEX idx_users_email_hash ON users (hashtext(email));
```

```js
await pool.query(
  "SELECT * FROM users WHERE hashtext(email) = hashtext($1)",
  [email]
);
```

**Result:** Index size drops from 1.1 GB to ~290 MB, cache hit rate reaches 99.5%, TPS increases 40–60%.

### Covering Index

When a query only needs columns present in the index, PostgreSQL can serve it entirely from the index without touching the main table.

```sql
CREATE INDEX idx_users_covering ON users (user_id) INCLUDE (username, email);
```

Useful for read-heavy endpoints where the same small set of columns is always selected.

### Clustered vs. Non-Clustered

**Clustered:** Data rows are physically stored in index order. Only one per table (typically the primary key). Range scans are fast because rows are co-located on disk.

**Non-Clustered:** A separate structure that points to row locations. Multiple allowed per table. Does not affect physical row order — each lookup may require a separate heap fetch.

### When Indexes Help vs. Hurt

| Scenario | Recommendation |
|----------|----------------|
| `WHERE` / `JOIN` / `ORDER BY` on large tables | Index |
| Read-heavy workloads | Index |
| Write-heavy workloads (high INSERT/UPDATE rate) | Use caution — indexes slow writes |
| Low-cardinality columns (e.g., boolean, status) | Avoid — full scan is often cheaper |
| Redundant or unused indexes | Remove — add write overhead and storage bloat |

### Index Maintenance

Every `INSERT`, `UPDATE`, and `DELETE` updates all indexes on that table. Use `EXPLAIN ANALYZE` to verify index usage and identify unused indexes.

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = $1;
```

### GIN Index Maintenance

GIN indexes accumulate pending entries during bulk inserts. Without periodic maintenance, query time degrades significantly.

```sql
-- Manual
VACUUM ANALYZE articles;
```

```bash
# Nightly cron
vacuumdb --analyze-in-stages --verbose -d appdb -t articles
```

**Result:** Query time on a 15M-row table drops from 12 ms (post-bulk-insert) to 0.12 ms after vacuum — a 100× improvement.

---

## 4. Query Optimization

### Avoid Repeated Computation

```sql
-- Inefficient — function evaluated per column
SELECT pg_stat_tuple('orders').*;

-- Efficient — function evaluated once
SELECT * FROM pg_stat_tuple('orders');
```

**Result:** Query time drops from 3.8 s to 420 ms.

### FDW Batch Fetch Size

By default, Foreign Data Wrappers fetch rows in small batches. Increasing `fetch_size` dramatically reduces round-trips for large remote queries.

```sql
ALTER SERVER my_fdw OPTIONS (ADD fetch_size '50000');
```

**Result:** Query time on remote data drops from 52 s to 7–10 s.

---

## 5. Caching Strategies

| Layer | Latency |
|-------|---------|
| PostgreSQL (disk/buffer) | ~1 ms |
| In-memory cache (Redis) | ~100 ns |

### Cache Layers

**In-Process Cache** — stored inside the application server process (e.g., Node.js LRU cache). Fastest possible access with no network call, but not shared across server instances.

**Distributed Cache (Redis/Memcached)** — shared across all instances via network. Slightly higher latency than in-process, but consistent across the fleet.

**CDN** — caches static assets and cacheable API responses at edge nodes close to users. First request fetches from origin; subsequent requests are served from the edge.

**Client Cache** — stored on the user's device (browser cache, service workers, mobile local storage). Private to that user; reduces server load for repeat visits without benefiting other users.

### Write Patterns

**Cache-Aside** *(most common)*
Application checks cache first. On miss, queries DB and populates cache. Cache is filled only with data that is actually requested.

**Write-Through**
Write to cache and DB atomically before confirming to the client. Strong consistency, higher write latency. Risk: if the DB write fails after the cache write, a rollback is needed.

**Write-Behind (Write-Back)**
Write to cache immediately, persist to DB asynchronously. Highest write throughput, but risk of data loss if the cache node fails before the flush completes.

**Read-Through**
Cache layer manages DB fallback transparently on misses. Simplifies application logic — often implemented via Redis modules or caching libraries like `cache-manager`.

### Eviction Policies

| Policy | Behavior | Use Case |
|--------|----------|----------|
| LRU (Least Recently Used) | Evicts items not accessed recently | General purpose |
| LFU (Least Frequently Used) | Evicts items accessed least often | Skewed access patterns |
| FIFO (First In, First Out) | Evicts the oldest entry | Simple time-based expiry |
| TTL (Time To Live) | Evicts after a fixed duration | Session tokens, API responses |

### Failure Modes

**Cache Stampede:** A key expires and many concurrent requests simultaneously miss and hit the DB.
*Fix:* Distributed lock on cache population, or probabilistic early recomputation (recompute slightly before TTL expires based on a probability function).

**Hot Key:** A single cache key receives disproportionate traffic (e.g., a trending post).
*Fix:* Shard the key across multiple nodes, or use a local in-process fallback cache to absorb read spikes before they reach Redis.

**Cache Consistency:** Cache and DB diverge after a write.
*Fix:* Invalidate on write (not on read), use atomic operations, or bound staleness with short TTLs. Avoid updating cache and DB in two separate non-atomic steps.

---

## 6. Partitioning

Partitioning splits a large table into smaller, independently-scanned segments while keeping them logically unified under a single table name.

### Horizontal Partitioning (by rows)

Splits data by row ranges — each partition holds a subset of rows with the same schema. Most common pattern for time-series and event data.

```sql
CREATE TABLE orders (
  order_id   BIGSERIAL,
  order_date DATE NOT NULL,
  revenue    NUMERIC(12,2) NOT NULL
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2024 PARTITION OF orders
  FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_2025 PARTITION OF orders
  FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

A query with `WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01'` scans only `orders_2024`, skipping all other partitions entirely (partition pruning).

**Use case:** Time-series data, analytics on historical records, archiving old data by dropping entire partitions instead of issuing bulk `DELETE`.

### Vertical Partitioning (by columns)

Splits data by columns — frequently accessed columns stay in the main table, large or rarely accessed columns move to a separate table joined by primary key.

```sql
-- Hot path: frequently queried
CREATE TABLE orders (
  order_id   BIGINT PRIMARY KEY,
  order_date DATE,
  revenue    NUMERIC
);

-- Cold path: rarely queried
CREATE TABLE orders_details (
  order_id    BIGINT REFERENCES orders,
  customer_id BIGINT,
  notes       TEXT
);
```

**Use case:** Tables with large `TEXT` / `JSONB` / `BYTEA` columns that inflate row size and pollute the buffer cache for queries that never touch those columns.

---

## 7. Sharding

Sharding distributes data across multiple independent database nodes (shards), each owning a portion of the total dataset. Each shard operates as a fully independent database.

### Horizontal Sharding (by rows)

All shards share the same schema. Rows are routed to a shard based on a shard key.

| Shard | Key Range |
|-------|-----------|
| db1 | `user_id` 1 – 1,000,000 |
| db2 | `user_id` 1,000,001 – 2,000,000 |

### Vertical Sharding (by domain)

Different data domains live on separate databases. Naturally aligns with microservice boundaries.

| Shard | Data |
|-------|------|
| db-users | `user_id`, `username`, `profile_picture` |
| db-messages | `message_id`, `content`, `attachments` |

### Shard Key Selection

A good shard key distributes load evenly and aligns with common query patterns.

| Good | Bad |
|------|-----|
| `user_id` (high cardinality, even distribution) | Sequential auto-increment IDs (all new writes hit one shard) |
| `hashtext(user_id)` | A low-cardinality field like `status` |
| `region` (if queries are always region-scoped) | A timestamp used alone |

### Hotspot Problem

When a single shard receives disproportionate traffic, fixes include hashing the shard key, replicating hot data to read replicas, or using a local in-process cache to absorb read traffic before it reaches the DB.

### Shard Lookup Service

A routing layer maps shard keys to physical shard locations. Without this, application code must embed routing logic directly and becomes hard to change when shards are rebalanced.

| Key Range | Shard Host |
|-----------|------------|
| 1 – 1,000,000 | `db1.internal` |
| 1,000,001 – 2,000,000 | `db2.internal` |

Tools: Vitess (MySQL/YouTube), Citus (PostgreSQL), application-level routing middleware.

### Cross-Shard Queries

Aggregations across all shards (e.g., total active users) require fan-out queries to every shard and client-side result merging. This is expensive. Mitigate by maintaining pre-aggregated summary tables updated asynchronously.

### Rebalancing

When a shard grows too large, data must be migrated to new nodes. Consistent hashing minimizes data movement — only keys near the new node's boundary need to move.

### Trade-offs

| Advantage | Disadvantage |
|-----------|--------------|
| Horizontal scale with traffic | Cross-shard transactions are complex |
| Reduced per-node load | Operational overhead increases significantly |
| Parallel query processing | Requires careful shard key design upfront |

---

## 8. Transactions & Consistency

### ACID

| Property | Meaning |
|----------|---------|
| Atomicity | All operations succeed or all are rolled back |
| Consistency | DB moves from one valid state to another |
| Isolation | Concurrent transactions don't interfere |
| Durability | Committed data survives crashes |

### Isolation Levels (PostgreSQL)

| Level | Behavior | Risk if Lower |
|-------|----------|---------------|
| Read Uncommitted | Reads uncommitted data | Dirty reads |
| Read Committed | Default. Sees only committed rows | Non-repeatable reads |
| Repeatable Read | Snapshot at transaction start | Phantom reads |
| Serializable | Full isolation | Highest contention, lowest throughput |

### Concurrency Control

**Pessimistic Locking** — locks the row for the duration of the transaction. No other transaction can modify it until the lock is released. Safe, but reduces throughput. Used in financial systems where conflicts are frequent.

```sql
SELECT * FROM accounts WHERE id = $1 FOR UPDATE;
```

**Optimistic Locking** — no lock acquired upfront; conflict is detected at commit time via a version column. High throughput; suitable when conflicts are rare (e.g., social media).

```sql
UPDATE posts
SET content = $1, version = version + 1
WHERE id = $2 AND version = $3;
-- 0 rows updated → conflict detected → retry at application layer
```

### Production Transaction Pattern (Node.js)

```js
const client = await pool.connect();

try {
  await client.query("BEGIN");

  await client.query(
    "INSERT INTO orders (user_id, total) VALUES ($1, $2)",
    [userId, total]
  );
  await client.query(
    "UPDATE inventory SET stock = stock - 1 WHERE product_id = $1",
    [productId]
  );

  await client.query("COMMIT");
} catch (err) {
  await client.query("ROLLBACK");
  throw err;
} finally {
  client.release();
}
```

### Distributed Transactions

When a transaction spans multiple services or databases, consistency requires a coordination protocol.

**Two-Phase Commit (2PC):**
1. Coordinator asks all participants to prepare and lock resources.
2. If all confirm readiness, coordinator sends commit. If any participant fails, coordinator sends rollback to all.

**Example — e-commerce order:**
- Order service creates the order record
- Payment service charges the card
- Inventory service decrements stock

If payment fails after the order is created, 2PC ensures all participants roll back, preventing orphaned records and double charges.

**Trade-off:** 2PC is synchronous and holds locks during the prepare phase, reducing throughput. For high-throughput systems, prefer eventual consistency via the **Saga pattern** — each service executes a local transaction and publishes an event; on failure, compensating transactions undo previous steps.

---

## 9. Security Best Practices

### SQL Injection Prevention (OWASP A03)

Always use parameterized queries. Never interpolate user input into query strings.

```js
// Safe
await pool.query(
  "SELECT * FROM users WHERE email = $1",
  [email]
);

// Never do this
await pool.query(`SELECT * FROM users WHERE email = '${email}'`);
```

### Additional Hardening

- Enforce TLS for all database connections (OWASP A02)
- Apply least-privilege IAM policies per service — no shared superuser credentials
- Enable PostgreSQL audit logging via `pgaudit`
- Enforce RBAC and row-level security where multi-tenancy applies (`ALTER TABLE ... ENABLE ROW LEVEL SECURITY`)
- Set transaction and lock timeouts to prevent runaway queries from holding locks (`statement_timeout`, `lock_timeout`)
- Log transaction failures for audit trails (required under PCI DSS for financial tables)
- Remove unused PostgreSQL extensions to reduce attack surface (OWASP A05)
- Validate and sanitize all inputs before they reach the DB layer

### NIST / PCI DSS Checklist

- Log retention strategy in place for sensitive tables
- No cardholder data stored unencrypted at rest
- Strict network boundaries between services accessing financial data
- All access to financial tables is logged and auditable

---

## Author

**Osama Alsrraj** — Full-Stack Enginner  
Focus: Performance, Scaling, Secure Systems

---

> Strong engineers do not only build systems — they continuously refine them for performance, scalability, and reliability.
