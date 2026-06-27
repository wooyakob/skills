# SQL++ Query Patterns

## Pagination

### Offset/Limit (simple, use for small result sets)

```sql
SELECT META().id, name, createdAt
FROM `bucket`.`scope`.`users`
WHERE type = "user" AND status = "active"
ORDER BY createdAt DESC
LIMIT 20 OFFSET 40;  -- page 3 (0-indexed)
```

```python
def paginate(cluster, page: int, page_size: int = 20):
    offset = page * page_size
    result = cluster.query(
        "SELECT META().id AS id, name FROM `b`.`s`.`users` "
        "WHERE type = 'user' ORDER BY name LIMIT $limit OFFSET $offset",
        QueryOptions(named_parameters={"limit": page_size, "offset": offset})
    )
    return list(result)
```

### Keyset Pagination (efficient for deep pages)

```sql
-- First page
SELECT META().id AS id, name, createdAt
FROM `bucket`.`scope`.`posts`
WHERE type = "post"
ORDER BY createdAt DESC, META().id ASC
LIMIT 20;

-- Next page (pass last row's createdAt and id as cursor)
SELECT META().id AS id, name, createdAt
FROM `bucket`.`scope`.`posts`
WHERE type = "post"
  AND (createdAt < $lastCreatedAt
       OR (createdAt = $lastCreatedAt AND META().id > $lastId))
ORDER BY createdAt DESC, META().id ASC
LIMIT 20;
```

## UPSERT Pattern (Insert or Update)

```python
# Upsert via SQL++ — merge based on document key
result = cluster.query(
    """
    UPSERT INTO `bucket`.`scope`.`users` (KEY, VALUE)
    VALUES ($key, $doc)
    RETURNING META().id AS id
    """,
    QueryOptions(named_parameters={
        "key": "user::u1",
        "doc": {"type": "user", "name": "Alice", "score": 42}
    })
)
```

For single-document upserts, prefer KV `collection.upsert()` — it's much faster.

## Bulk INSERT from SELECT

```sql
-- Copy documents from one collection to another
INSERT INTO `bucket`.`scope`.`users_archive` (KEY, VALUE)
SELECT META().id AS _key, *
FROM `bucket`.`scope`.`users`
WHERE status = "inactive" AND lastLoginAt < "2023-01-01";
```

## UPDATE Patterns

```sql
-- Add or update a field
UPDATE `bucket`.`scope`.`products`
SET updatedAt = NOW_STR(), featured = true
WHERE META().id = "product::p1"
RETURNING META().id, featured;

-- Conditional update
UPDATE `bucket`.`scope`.`orders`
SET status = "shipped", shippedAt = NOW_STR()
WHERE META().id IN $ids AND status = "processing"
RETURNING META().id, status;

-- Update nested field
UPDATE `bucket`.`scope`.`users`
SET preferences.theme = "dark"
WHERE META().id = "user::u1";

-- Append to array in SQL++
UPDATE `bucket`.`scope`.`posts`
SET tags = ARRAY_APPEND(tags, "featured")
WHERE META().id = "post::p42";

-- Bulk update with WHERE
UPDATE `bucket`.`scope`.`products`
SET discountPct = 20, saleEndsAt = "2024-03-31"
WHERE category = "electronics" AND price > 100;
```

## Aggregation Patterns

```sql
-- Group by with HAVING
SELECT
    category,
    COUNT(*) AS productCount,
    ROUND(AVG(price), 2) AS avgPrice,
    MIN(price) AS minPrice,
    MAX(price) AS maxPrice,
    SUM(CASE WHEN stock > 0 THEN 1 ELSE 0 END) AS inStockCount
FROM `bucket`.`scope`.`products`
WHERE type = "product"
GROUP BY category
HAVING COUNT(*) >= 5
ORDER BY productCount DESC;

-- Pivot-style aggregation
SELECT
    SUM(CASE WHEN MONTH(createdAt) = 1 THEN total ELSE 0 END) AS jan,
    SUM(CASE WHEN MONTH(createdAt) = 2 THEN total ELSE 0 END) AS feb,
    SUM(CASE WHEN MONTH(createdAt) = 3 THEN total ELSE 0 END) AS mar
FROM `bucket`.`scope`.`orders`
WHERE type = "order" AND YEAR(createdAt) = 2024;

-- Nested aggregation (count per sub-group)
SELECT
    status,
    ARRAY_AGG({"id": META().id, "total": total}) AS orders,
    SUM(total) AS statusTotal
FROM `bucket`.`scope`.`orders`
WHERE type = "order"
GROUP BY status;
```

## Full Collection Scan (avoid in production)

```sql
-- If no index exists, this does a primary scan — add an index
SELECT COUNT(*) FROM `bucket`.`scope`.`users`;

-- Force index hint (use sparingly)
SELECT * FROM `bucket`.`scope`.`users`
USE INDEX (idx_users_status USING GSI)
WHERE status = "active";
```

## MERGE Statement (Upsert from another source)

```sql
MERGE INTO `bucket`.`scope`.`inventory` AS target
USING (
    SELECT META().id AS productId, stockDelta
    FROM `bucket`.`scope`.`shipments`
    WHERE processed = false
) AS source
ON META(target).id = source.productId
WHEN MATCHED THEN
    UPDATE SET target.stock = target.stock + source.stockDelta,
               target.updatedAt = NOW_STR()
WHEN NOT MATCHED THEN
    INSERT (KEY source.productId, VALUE {"stock": source.stockDelta, "type": "inventory"});
```

## Query Error Handling (Python)

```python
from couchbase.exceptions import (
    QueryException, PlanningFailureException,
    IndexFailureException, CouchbaseException
)

try:
    result = cluster.query(sql, QueryOptions(named_parameters=params))
    rows = list(result)
except PlanningFailureException as e:
    # Query could not be planned — likely missing index
    print("Index missing:", e)
except IndexFailureException as e:
    # Index error during execution
    print("Index error:", e)
except QueryException as e:
    # General query error
    print("Query error:", e.message)
```

## Query Metrics

```python
from couchbase.options import QueryOptions

result = cluster.query(sql, QueryOptions(metrics=True))
rows = list(result)
metrics = result.metadata().metrics()
print(f"Elapsed: {metrics.elapsed_time()}")
print(f"Execution: {metrics.execution_time()}")
print(f"Result count: {metrics.result_count()}")
print(f"Mutation count: {metrics.mutation_count()}")
```

## Analytics Queries

Couchbase Analytics (Columnar) uses the same SQL++ dialect with minor differences:

```python
# Use cluster.analytics_query() instead of cluster.query()
result = cluster.analytics_query(
    "SELECT category, COUNT(*) AS cnt FROM `my-bucket`.`scope`.`products` GROUP BY category",
)
for row in result:
    print(row)
```

Analytics supports: `USE LINKS`, `CREATE DATASET`, `CREATE ANALYTICS COLLECTION`, window functions, complex aggregations without index requirements.
