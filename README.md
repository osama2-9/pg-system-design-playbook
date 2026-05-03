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
7. [Sharding](#7-sharding-horizontal-scaling)
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

**Fix:** Co-locate services in the same AZ, or use Unix domain sockets for local connections.

```js
import { Pool } from "pg";

const pool = new Pool({
  host: "/var/run/postgresql",
  database: "app",
});
```

**Result:** Latency drops to ~1.5 ms. Throughput increases significantly with no infrastructure changes.

---

## 2. PostgreSQL Storage Optimization

### Column Alignment

PostgreSQL pads columns to align with type boundaries. Reordering columns by descending size reduces padding and improves storage efficiency.

```sql
-- Inefficient layout
CREATE TABLE logs (
  message    TEXT,
  created_at TIMESTAMPTZ,
  status     INT,
  user_id    BIGINT
);

-- Optimized layout
CREATE TABLE logs_opt (
  created_at TIMESTAMPTZ,
  user_id    BIGINT,
  status     INT,
  message    TEXT
);
```

**Result:** ~12% storage reduction, better buffer cache utilization.

### Zero-Cost Column Addition

```sql
ALTER TABLE users
ADD COLUMN is_verified BOOLEAN DEFAULT false;
```

No table rewrite. Instant migration. Safe under high traffic.

---

## 3. Indexing Strategies

### Functional Index for Hash-Based Lookups

```sql
CREATE INDEX idx_users_email_hash
ON users (hashtext(email));
```

```js
await pool.query(
  "SELECT * FROM users WHERE hashtext(email) = hashtext($1)",
  [email]
);
```

**Result:** Smaller index, higher cache hit rate, improved read throughput.

### When Indexes Help vs. Hurt

| Scenario | Use Index? |
|----------|------------|
| WHERE / JOIN / ORDER BY on large tables | Yes |
| Read-heavy workloads | Yes |
| Write-heavy workloads | Caution |
| Excessive or redundant indexes | No — adds write overhead |

---

## 4. Query Optimization

### Avoid Repeated Computation

```sql
-- Inefficient
SELECT pg_stat_tuple('orders').*;

-- Efficient
SELECT * FROM pg_stat_tuple('orders');
```

### FDW Batch Fetch Size

```sql
ALTER SERVER my_fdw OPTIONS (ADD fetch_size '50000');
```

Dramatically reduces round-trips when querying remote data sources via Foreign Data Wrappers.

---

## 5. Caching Strategies

| Layer | Latency |
|-------|---------|
| PostgreSQL (disk/buffer) | ~1 ms |
| In-memory cache (Redis) | ~100 µs |

### Common Patterns

**Cache-Aside** *(most common)*
Check cache → on miss, query DB → store result in cache.

**Write-Through**
Write to cache and DB atomically. Strong consistency, higher write latency.

**Write-Behind**
Write to cache, persist to DB asynchronously. High throughput, risk of data loss on failure.

### Failure Modes to Know

**Cache Stampede:** Many requests simultaneously miss cache and hammer the DB.
*Fix:* Use a distributed lock or probabilistic early expiration.

**Hot Key:** A single cache key receives disproportionate traffic.
*Fix:* Key sharding or local replica caching.

---

## 6. Partitioning

Split large tables into smaller, independently-scanned partitions.

```sql
CREATE TABLE orders (
  order_id   BIGSERIAL,
  order_date DATE,
  revenue    NUMERIC
) PARTITION BY RANGE (order_date);
```

Queries with date filters scan only the relevant partition, not the full table. Critical for time-series and analytics workloads.

---

## 7. Sharding (Horizontal Scaling)

Distribute data across multiple independent database nodes.

| Shard | Range |
|-------|-------|
| A | users 1 – 1,000,000 |
| B | users 1,000,001 – 2,000,000 |

### Challenges

- Cross-shard queries require application-level aggregation
- Rebalancing shards is operationally expensive
- Shard key selection is critical — poor choice leads to hotspots

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

| Level | Behavior |
|-------|----------|
| Read Committed | Default. Sees only committed data. |
| Repeatable Read | Snapshot at transaction start. |
| Serializable | Full isolation. Highest contention. |

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

- Enforce TLS for all database connections
- Apply least-privilege IAM policies per service
- Enable PostgreSQL audit logging (`pgaudit`)
- Enforce RBAC at the application layer
- Validate and sanitize all inputs before they reach the DB layer

---

## Author

**Osama Alsrraj** — Backend & Distributed Systems Engineer  
Focus: Performance, Scaling, Secure Systems

---

> Strong engineers do not only build systems — they continuously refine them for performance, scalability, and reliability.
