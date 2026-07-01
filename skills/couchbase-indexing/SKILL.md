---
name: couchbase-indexing
description: Create and manage Couchbase Global Secondary Indexes (GSI) for SQL++ queries. Use when creating indexes to speed up queries, designing composite or partial indexes, building covering indexes, analyzing query plans with EXPLAIN, or managing index lifecycle (build, drop, monitor).
---

# Couchbase Indexing

Couchbase uses **Global Secondary Indexes (GSI)** to support SQL++ queries. Without an index, queries require a full collection scan (slow and resource-intensive). Good index design is the single biggest lever for query performance.

## Index Basics

```sql
-- Primary index (use ONLY in development — full scan every query)
CREATE PRIMARY INDEX ON `bucket`.`scope`.`collection`;

-- Secondary index on a single field
CREATE INDEX idx_users_email ON `bucket`.`scope`.`users`(email);

-- Secondary index with a WHERE clause to query by
CREATE INDEX idx_users_type_email
ON `bucket`.`scope`.`users`(email)
WHERE type = "user";

-- Drop an index
DROP INDEX `idx_users_email` ON `bucket`.`scope`.`users`;
```

## Composite Indexes (Multi-Field)

Order fields by: **equality filters first, then range/sort fields**.

```sql
-- Query: WHERE type = "order" AND status = "pending" AND createdAt > "2024-01-01"
CREATE INDEX idx_orders_type_status_created
ON `bucket`.`scope`.`orders`(type, status, createdAt);

-- Query: WHERE category = "electronics" ORDER BY price ASC
CREATE INDEX idx_products_cat_price
ON `bucket`.`scope`.`products`(category, price);
```

## Partial Indexes (Filtered)

Index only documents matching a predicate — smaller, faster, cheaper:

```sql
-- Only index active users (not archived/deleted)
CREATE INDEX idx_active_users_email
ON `bucket`.`scope`.`users`(email, name)
WHERE type = "user" AND status = "active";

-- Only index the last 90 days of events
CREATE INDEX idx_recent_events
ON `bucket`.`scope`.`events`(userId, eventType, timestamp)
WHERE timestamp >= DATE_ADD_STR(NOW_STR(), -90, "day");
```

## Covering Indexes

A **covering index** includes all fields the query needs, eliminating a fetch to the data node:

```sql
-- Query: SELECT email, name, status FROM users WHERE type = "user" AND status = "active"
-- This index "covers" the query — no document fetch needed
CREATE INDEX idx_users_covering
ON `bucket`.`scope`.`users`(type, status, email, name)
WHERE type = "user";
```

Add all SELECT and WHERE fields to the index definition. Covered queries are significantly faster.

## Array Indexes

Index individual elements within arrays:

```sql
-- Documents: { "tags": ["sale", "new", "electronics"] }
CREATE INDEX idx_products_tags
ON `bucket`.`scope`.`products`(DISTINCT ARRAY t FOR t IN tags END);

-- Query using the array index:
SELECT name, price FROM `bucket`.`scope`.`products`
WHERE ANY t IN tags SATISFIES t = "sale" END;
```

## Adaptive Index (Flexible Schema)

Indexes all non-null fields in a document — useful when schema varies widely:

```sql
CREATE INDEX idx_adaptive ON `bucket`.`scope`.`collection`(
  DISTINCT PAIRS(self)
)
WHERE type = "product";
```

## Checking Index Status

```sql
-- List all indexes
SELECT name, state, progress
FROM system:indexes
WHERE keyspace_id = "collection-name";

-- Check a specific index
SELECT * FROM system:indexes WHERE name = "idx_users_email";
```

States: `online` (ready), `building` (deferred build in progress), `pending` (waiting for build trigger).

## Deferred Builds (Bulk Indexing)

For large datasets, create all indexes as deferred, then build in a single pass:

```sql
-- Create with DEFER BUILD
CREATE INDEX idx_a ON `bucket`.`scope`.`users`(email)
WITH { "defer_build": true };

CREATE INDEX idx_b ON `bucket`.`scope`.`users`(status)
WITH { "defer_build": true };

-- Trigger build for all deferred indexes on this keyspace
BUILD INDEX ON `bucket`.`scope`.`users`(idx_a, idx_b);
```

## EXPLAIN — Understand Query Plans

```sql
EXPLAIN
SELECT name, email FROM `bucket`.`scope`.`users`
WHERE type = "user" AND status = "active"
LIMIT 10;
```

Key things to look for in the output:
- `indexScan` — good: using an index
- `primaryScan` — bad: full scan (create an index)
- `fetch` — document fetch after index scan (use a covering index to eliminate)
- `filter` — residual filter applied after index scan (add more fields to the index)

## Index Advisor

Let the query engine suggest indexes:

```sql
SELECT * FROM system:my_user_info;

-- Run the advisor
SELECT ADVISOR(
  "SELECT email, name FROM `bucket`.`scope`.`users` WHERE type = 'user' AND status = 'active'",
  "SELECT * FROM `bucket`.`scope`.`orders` WHERE userId = $id AND status = 'pending'"
);
```

For index patterns for specific query shapes and best practices for production, see `references/index-patterns.md`.
