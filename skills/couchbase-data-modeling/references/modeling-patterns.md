# Couchbase Document Modeling Patterns

## Many-to-Many Relationships

Use a junction/edge document instead of embedded arrays (when either side can be large):

```json
// user::u1
{ "type": "user", "name": "Alice" }

// group::g1
{ "type": "group", "name": "Engineering" }

// membership::u1::g1  (junction document)
{
  "type": "membership",
  "userId": "user::u1",
  "groupId": "group::g1",
  "role": "member",
  "joinedAt": "2024-01-15T00:00:00Z"
}
```

Query memberships:
```sql
SELECT u.name, m.role
FROM `bucket`.`scope`.`memberships` AS m
JOIN `bucket`.`scope`.`users` AS u ON META(u).id = m.userId
WHERE m.groupId = "group::g1";
```

## Hierarchical / Tree Structures

### Adjacency List (simple, works for small trees)
```json
{ "type": "category", "id": "cat-001", "name": "Electronics", "parentId": null }
{ "type": "category", "id": "cat-002", "name": "Audio", "parentId": "cat-001" }
{ "type": "category", "id": "cat-003", "name": "Headphones", "parentId": "cat-002" }
```

### Materialized Path (fast subtree queries)
```json
{ "type": "category", "id": "cat-003", "name": "Headphones", "path": "cat-001/cat-002/cat-003" }
```
```sql
-- All descendants of Electronics (cat-001)
SELECT name FROM `b`.`s`.`categories`
WHERE path LIKE "cat-001/%";
```

### Nested Sets (for read-heavy hierarchies)
Store `lft` and `rgt` values; subtree query is a range scan:
```json
{ "type": "category", "name": "Electronics", "lft": 1, "rgt": 10 }
{ "type": "category", "name": "Audio",       "lft": 2, "rgt": 7 }
```
```sql
SELECT * FROM categories WHERE lft BETWEEN 1 AND 10;  -- all under Electronics
```

## Multi-Tenancy

### Scope-per-tenant (strong isolation, Capella Enterprise)
```
my-bucket
├── tenant-acme   → users, orders, products
├── tenant-bigco  → users, orders, products
```
Pros: clean isolation, separate indexes, independent access control.
Cons: more scopes to manage, limited by Couchbase scope limits.

### Field-per-tenant (simple, flexible)
```json
{ "type": "order", "tenantId": "acme", "orderId": "001", ... }
```
Always include `tenantId` in every index:
```sql
CREATE INDEX idx_orders_tenant ON `b`.`s`.`orders`(tenantId, status, createdAt);
SELECT * FROM `b`.`s`.`orders` WHERE tenantId = "acme" AND status = "pending";
```

### Key-prefix-per-tenant
```
acme::order::001
bigco::order::001
```
Fast KV lookup by tenant without queries, but forces all queries to filter by prefix.

## Time-Series Data

```json
{
  "type": "metric",
  "deviceId": "device-001",
  "metric": "temperature",
  "value": 23.5,
  "timestamp": "2024-03-15T14:32:01.123Z",
  "tags": { "location": "server-room-a", "unit": "celsius" }
}
```

Key: `metric::device-001::2024-03-15::uuid4` — use TTL for auto-expiry.

Rollup: Use Eventing to aggregate minute-level into hourly/daily summaries.

**Bucket rollup pattern** (aggregate recent raw into period docs):
```json
{
  "type": "metric_hourly",
  "deviceId": "device-001",
  "metric": "temperature",
  "hour": "2024-03-15T14:00:00Z",
  "min": 22.1, "max": 24.8, "avg": 23.3, "count": 3600
}
```

## E-Commerce Catalog

```json
// Product base
{
  "type": "product",
  "sku": "HDPH-001",
  "name": "Wireless Headphones",
  "category": ["electronics", "audio"],
  "brand": "AudioMax",
  "basePrice": 149.99,
  "description": "...",
  "specs": {
    "batteryLife": "30h",
    "connectivity": "Bluetooth 5.0",
    "weight": "250g"
  },
  "variants": [
    { "variantSku": "HDPH-001-BLK", "color": "black", "stock": 42 },
    { "variantSku": "HDPH-001-WHT", "color": "white", "stock": 18 }
  ],
  "images": ["https://...", "https://..."],
  "tags": ["wireless", "noise-cancelling", "sale"],
  "status": "active",
  "createdAt": "2024-01-01T00:00:00Z"
}

// Inventory (separate for high-write scenarios)
{
  "type": "inventory",
  "sku": "HDPH-001",
  "stock": 60,
  "reserved": 5,
  "updatedAt": "2024-03-15T14:00:00Z"
}
```

## Event Sourcing Pattern

Store immutable events; derive state by replaying:

```json
{
  "type": "event",
  "aggregateId": "order::ord-001",
  "aggregateType": "order",
  "eventType": "OrderPlaced",
  "version": 1,
  "timestamp": "2024-03-15T10:00:00Z",
  "payload": { "userId": "user::u1", "total": 99.99 }
}
{
  "type": "event",
  "aggregateId": "order::ord-001",
  "aggregateType": "order",
  "eventType": "OrderShipped",
  "version": 2,
  "timestamp": "2024-03-16T08:00:00Z",
  "payload": { "trackingNumber": "1Z999AA1" }
}
```

Store materialized views as separate documents, updated via Eventing functions.

## Large Document Splitting

When a document exceeds ~1 MB (or its hot fields are much smaller than its body):

```json
// Header doc — frequently accessed fields (hot)
{
  "type": "article",
  "id": "art-001",
  "title": "Getting Started with Couchbase",
  "author": "Alice",
  "publishedAt": "2024-03-15T00:00:00Z",
  "tags": ["tutorial", "couchbase"],
  "bodyRef": "article_body::art-001"  // pointer to body doc
}

// Body doc — rarely accessed
{
  "type": "article_body",
  "articleId": "art-001",
  "body": "... (long markdown content) ..."
}
```

Fetch header with KV; conditionally fetch body only when displaying the full article.

## Schema Evolution

```python
CURRENT_VERSION = 3

def migrate(doc: dict) -> dict:
    v = doc.get("schemaVersion", 1)
    if v < 2:
        # v1 → v2: flatten name fields
        doc["name"] = f"{doc.pop('firstName', '')} {doc.pop('lastName', '')}".strip()
        doc["schemaVersion"] = 2
    if v < 3:
        # v2 → v3: add status field
        doc.setdefault("status", "active")
        doc["schemaVersion"] = 3
    return doc

def get_user(collection, key: str) -> dict:
    result = collection.get(key)
    doc = result.content_as[dict]
    if doc.get("schemaVersion", 1) < CURRENT_VERSION:
        doc = migrate(doc)
        collection.replace(key, doc)  # lazy write-back
    return doc
```
