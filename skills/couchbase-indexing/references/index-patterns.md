# Index Design Patterns

## Index Advisor

Let the query engine suggest indexes for your workload:

```sql
-- Run for a set of queries
SELECT ADVISOR([
  "SELECT name, email FROM `b`.`s`.`users` WHERE type = 'user' AND status = 'active'",
  "SELECT * FROM `b`.`s`.`orders` WHERE userId = 'user::u1' AND status = 'pending'",
  "SELECT category, COUNT(*) FROM `b`.`s`.`products` WHERE type = 'product' GROUP BY category"
]);
```

The advisor returns recommended indexes. Review and validate before creating.

## Index Field Ordering Rules

The optimizer uses a "leading key" strategy — fields must match from left to right:

```sql
CREATE INDEX idx ON `b`.`s`.`users`(status, country, createdAt);

-- Uses idx (leading key status matches)
SELECT * FROM users WHERE status = "active";

-- Uses idx (leading keys status + country match)
SELECT * FROM users WHERE status = "active" AND country = "US";

-- Uses idx (all three keys)
SELECT * FROM users WHERE status = "active" AND country = "US" AND createdAt > "2024-01-01";

-- Does NOT use idx (status is missing from WHERE)
SELECT * FROM users WHERE country = "US";
```

**Rule**: Put equality fields first, range/sort fields last.

## Covering Index Checklist

A covering index eliminates the "fetch" step. Include every field in:
1. `WHERE` clause
2. `SELECT` clause
3. `ORDER BY` clause
4. `JOIN` ON clause

```sql
-- Query
SELECT email, name, createdAt
FROM `b`.`s`.`users`
WHERE type = "user" AND status = "active"
ORDER BY createdAt DESC
LIMIT 20;

-- Covering index (includes all SELECT + WHERE + ORDER BY fields)
CREATE INDEX idx_users_covering
ON `b`.`s`.`users`(type, status, createdAt DESC, email, name)
WHERE type = "user";
```

## Partial Index Patterns

### Type-filtered (most common)
```sql
CREATE INDEX idx_orders_pending
ON `b`.`s`.`orders`(userId, createdAt)
WHERE type = "order" AND status = "pending";
```

### Time-windowed (rolling retention)
```sql
-- Only index events from last 30 days (re-created or updated periodically)
CREATE INDEX idx_recent_events
ON `b`.`s`.`events`(userId, eventType, timestamp)
WHERE timestamp >= DATE_ADD_STR(NOW_STR(), -30, "day");
```

### Non-null filter (sparse index)
```sql
-- Only index documents with a premiumSince field (most users don't have it)
CREATE INDEX idx_premium_users
ON `b`.`s`.`users`(premiumSince, userId)
WHERE premiumSince IS NOT MISSING AND premiumSince IS NOT NULL;
```

## Array Index Patterns

```sql
-- Simple array of strings: { "tags": ["sale", "new"] }
CREATE INDEX idx_tags
ON `b`.`s`.`products`(DISTINCT ARRAY t FOR t IN tags END);

-- Query:
SELECT name FROM `b`.`s`.`products`
WHERE ANY t IN tags SATISFIES t = "sale" END;

-- Array of objects: { "lineItems": [{ "productId": "p1", "qty": 2 }] }
CREATE INDEX idx_order_products
ON `b`.`s`.`orders`(DISTINCT ARRAY item.productId FOR item IN lineItems END);

-- Query:
SELECT META().id FROM `b`.`s`.`orders`
WHERE ANY item IN lineItems SATISFIES item.productId = "product::p1" END;

-- Array with filter: only index items above threshold
CREATE INDEX idx_expensive_items
ON `b`.`s`.`orders`(
  DISTINCT ARRAY item.productId FOR item IN lineItems WHEN item.price > 100 END
);
```

## Multi-Collection Index (function-based)

Index a computed/derived value:

```sql
-- Index the month of creation for partitioned reporting
CREATE INDEX idx_orders_month
ON `b`.`s`.`orders`(DATE_FORMAT_STR(createdAt, "2006-01"), status);

-- Query:
SELECT * FROM `b`.`s`.`orders`
WHERE DATE_FORMAT_STR(createdAt, "2006-01") = "2024-03"
  AND status = "completed";

-- Index the length of an array
CREATE INDEX idx_order_size
ON `b`.`s`.`orders`(ARRAY_LENGTH(lineItems))
WHERE type = "order";
```

## Index for JOIN Operations

Ensure the right side of a JOIN has an index on the join key:

```sql
-- Query
SELECT o.id, o.total, u.name
FROM `b`.`s`.`orders` AS o
JOIN `b`.`s`.`users` AS u ON META(u).id = o.userId
WHERE o.status = "pending";

-- Index needed on orders (driving side)
CREATE INDEX idx_orders_pending ON `b`.`s`.`orders`(status, userId)
WHERE type = "order" AND status = "pending";

-- Index needed on users (probe side) — primary or on META().id (automatic)
-- Usually users is probed by primary key, so no extra index needed
```

## Monitoring Index Health

```sql
-- All indexes with status and progress
SELECT name, state, keyspace_id, index_key, condition,
       progress, num_docs_indexed, num_docs_pending
FROM system:indexes
ORDER BY state, name;

-- Indexes not yet online
SELECT name, state FROM system:indexes WHERE state != "online";

-- Index usage stats (requires index stats enabled)
SELECT * FROM system:index_stats
WHERE keyspace_id = "users";
```

## Dropping Redundant Indexes

Look for indexes where one is a prefix of another:

```sql
-- idx_a covers idx_b if idx_b's fields are a leading prefix of idx_a's fields
-- and idx_a's condition (WHERE) is equal or more restrictive

-- Example: if you have both of these, idx_narrow may be redundant
-- idx_wide: CREATE INDEX ON users(type, status, email, name) WHERE type = "user"
-- idx_narrow: CREATE INDEX ON users(type, status) WHERE type = "user"
-- → idx_wide covers everything idx_narrow does; drop idx_narrow
```

## Index Replicas (Production HA)

```sql
CREATE INDEX idx_users_email
ON `b`.`s`.`users`(email)
WITH { "num_replica": 1 };  -- 1 replica = 2 total copies
```

Replicas:
- Provide high availability (index remains available if one index node fails)
- Enable load balancing across index nodes
- Increase memory usage proportionally

## BUILD INDEX After Deferred Creation

```sql
-- Create indexes without immediately building them
CREATE INDEX idx_a ON `b`.`s`.`users`(email) WITH {"defer_build": true};
CREATE INDEX idx_b ON `b`.`s`.`users`(name) WITH {"defer_build": true};
CREATE INDEX idx_c ON `b`.`s`.`users`(status, createdAt) WITH {"defer_build": true};

-- Build all deferred indexes in a single scan pass
BUILD INDEX ON `b`.`s`.`users`(idx_a, idx_b, idx_c);

-- Monitor build progress
SELECT name, state, progress FROM system:indexes
WHERE keyspace_id = "users" AND state = "building";
```

One BUILD INDEX pass is much more efficient than three separate creates for large collections.
