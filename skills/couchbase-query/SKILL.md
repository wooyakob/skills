---
name: couchbase-query
description: Write and execute N1QL/SQL++ queries against Couchbase. Use when searching for documents by field values, performing aggregations, doing joins across collections, or running analytical queries. Covers SELECT, INSERT, UPDATE, DELETE, UPSERT, parameters, scan consistency, and query optimization hints.
allowed-tools: Bash, Read, Write, Edit
---

# Couchbase N1QL / SQL++ Queries

N1QL (Non-First Normal Form Query Language) is Couchbase's SQL dialect — officially called SQL++ — that queries JSON documents. It runs via `cluster.query()` for cluster-wide queries or `scope.query()` for queries scoped to a specific scope (avoids fully-qualifying keyspaces).

## Keyspace Syntax

In SQL++, the collection path is the "keyspace":

```sql
-- Fully qualified (from cluster.query)
SELECT * FROM `bucket-name`.`scope-name`.`collection-name` WHERE ...

-- Scope-relative (from scope.query — omit bucket and scope)
SELECT * FROM `collection-name` WHERE ...
```

Backtick-quote names containing hyphens or reserved words.

## Running Queries (Python SDK v4.x)

```python
from couchbase.options import QueryOptions
from couchbase.n1ql import QueryScanConsistency

# Named parameters — preferred (prevents injection)
result = cluster.query(
    "SELECT META().id AS id, name, score FROM `my-bucket`.`_default`.`users` WHERE type = $type AND score > $min",
    QueryOptions(named_parameters={"type": "player", "min": 100})
)
for row in result:
    print(row["id"], row["name"], row["score"])

# Positional parameters
result = cluster.query(
    "SELECT * FROM `bucket`.`scope`.`coll` WHERE name = $1 AND age > $2",
    QueryOptions(positional_parameters=["Alice", 30])
)

# Scope-relative query (no bucket/scope prefix needed in FROM)
result = scope.query(
    "SELECT META().id, * FROM `users` WHERE type = $type",
    QueryOptions(named_parameters={"type": "admin"})
)
```

## Common Statements

```sql
-- SELECT with filter, sort, limit
SELECT META().id AS id, name, email, createdAt
FROM `bucket`.`scope`.`users`
WHERE type = "user" AND status = "active"
ORDER BY createdAt DESC
LIMIT 20 OFFSET 40;

-- INSERT a new document (with explicit key)
INSERT INTO `bucket`.`scope`.`orders` (KEY, VALUE)
VALUES ("order::12345", {
  "type": "order", "userId": "u1", "total": 99.99, "status": "pending"
});

-- UPSERT — create or replace
UPSERT INTO `bucket`.`scope`.`users` (KEY, VALUE)
VALUES ("user::u1", {"type": "user", "name": "Alice", "score": 42});

-- UPDATE specific fields
UPDATE `bucket`.`scope`.`orders`
SET status = "shipped", shippedAt = NOW_STR()
WHERE META().id = "order::12345";

-- DELETE with WHERE
DELETE FROM `bucket`.`scope`.`sessions`
WHERE type = "session" AND expiresAt < NOW_STR();
```

## Scan Consistency

Controls when query results reflect recent KV writes:

```python
from couchbase.n1ql import QueryScanConsistency

# NOT_BOUNDED — fastest, may miss very recent writes (default)
result = cluster.query(sql, QueryOptions(scan_consistency=QueryScanConsistency.NOT_BOUNDED))

# REQUEST_PLUS — waits for all pending KV mutations to be indexed
result = cluster.query(sql, QueryOptions(scan_consistency=QueryScanConsistency.REQUEST_PLUS))
```

Use `REQUEST_PLUS` after writes when you need to query what you just wrote. For analytics and reporting, `NOT_BOUNDED` is fine.

## Arrays and Nested Documents

```sql
-- ANY / EVERY — test array elements
SELECT name FROM `bucket`.`scope`.`products`
WHERE ANY tag IN tags SATISFIES tag = "sale" END;

-- EVERY — all elements must satisfy
SELECT name FROM `bucket`.`scope`.`orders`
WHERE EVERY item IN lineItems SATISFIES item.price > 0 END;

-- UNNEST — flatten array into rows (like a JOIN to each element)
SELECT p.name, t AS tag
FROM `bucket`.`scope`.`products` p
UNNEST p.tags AS t
WHERE t LIKE "elec%";

-- Nested field access
SELECT address.city, address.zip
FROM `bucket`.`scope`.`users`
WHERE address.country = "US";
```

## Aggregations

```sql
SELECT
  category,
  COUNT(*) AS total,
  AVG(price) AS avgPrice,
  SUM(stock) AS totalStock,
  MAX(price) AS maxPrice
FROM `bucket`.`scope`.`products`
WHERE type = "product"
GROUP BY category
HAVING COUNT(*) > 5
ORDER BY total DESC;
```

## JOIN Across Collections

```sql
-- ANSI JOIN (recommended)
SELECT o.id, o.total, u.name AS customerName
FROM `bucket`.`scope`.`orders` AS o
JOIN `bucket`.`scope`.`users` AS u ON o.userId = META(u).id
WHERE o.status = "pending";

-- USE KEYS join (older syntax, joins on document keys)
SELECT o.*, u.name
FROM `bucket`.`scope`.`orders` AS o
JOIN `bucket`.`scope`.`users` AS u USE KEYS o.userId;
```

## Query in Node.js

```javascript
const result = await cluster.query(
  'SELECT META().id AS id, name FROM `bucket`.`scope`.`users` WHERE type = $type',
  { parameters: { type: 'admin' } }
)
for (const row of result.rows) {
  console.log(row.id, row.name)
}
```

For full SQL++ syntax reference, pagination patterns, window functions, and prepared statements, see `references/sql-plus-plus.md` and `references/query-patterns.md`.
