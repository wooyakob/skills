---
name: couchbase-data-modeling
description: Design Couchbase document schemas and data models. Use when structuring JSON documents, deciding how to organize buckets/scopes/collections, choosing between embedded and referenced documents, designing document keys, or migrating a relational schema to Couchbase's document model.
---

# Couchbase Data Modeling

Couchbase stores JSON documents. Good modeling exploits the document structure to eliminate JOINs and serve common access patterns from a single KV lookup.

## Core Principles

1. **Model for your queries** — design documents around how you read data, not how you write it
2. **Embed for reads** — data always accessed together should live in the same document
3. **Reference for writes** — data updated independently should be separate documents
4. **One collection per entity type** — avoids type collisions and simplifies indexing
5. **Keep documents focused** — large documents increase memory pressure; target < 20 KB for hot data

## Document Structure Convention

Every document should include a `type` field — it enables polymorphic queries and debugging:

```json
{
  "type": "user",
  "id": "u1a2b3",
  "name": "Alice Chen",
  "email": "alice@example.com",
  "createdAt": "2024-03-15T10:00:00Z",
  "status": "active",
  "address": {
    "street": "123 Main St",
    "city": "San Francisco",
    "country": "US"
  },
  "tags": ["premium", "beta-tester"]
}
```

## Document Key Design

The key is the fastest way to retrieve a document. Make it meaningful and predictable:

```
{type}::{id}                →  user::u1a2b3
{type}::{tenant}::{id}      →  order::acme::00042
{type}::{date}::{sequence}  →  event::2024-03-15::000001
```

- Max 250 bytes
- Use `::`  as a separator (avoid spaces, `/`, `\`)
- The type prefix aids debugging and namespace management
- If using auto-generated IDs, prefer UUIDs or ULIDs (ULIDs sort chronologically)

## Bucket → Scope → Collection Organization

```
my-bucket
├── _default._default      (legacy / simple apps)
├── ecommerce
│   ├── users
│   ├── orders
│   ├── products
│   └── reviews
└── analytics
    ├── events
    └── sessions
```

**Rules of thumb:**
- **Bucket** — one per major domain or environment (prod/staging) — subject to resource limits
- **Scope** — one per application or team within a bucket
- **Collection** — one per entity type; use as the unit of indexing and access control

## Embed vs Reference

### Embed when:
- The nested data is **always accessed with the parent** (e.g., address on a user)
- The nested data has **no independent identity** (e.g., line items on an order)
- The nested collection is **small and bounded** (e.g., ≤ 20 items)

```json
{
  "type": "order",
  "id": "ord-001",
  "userId": "user::u1",
  "lineItems": [
    { "productId": "prod::p1", "qty": 2, "price": 29.99 },
    { "productId": "prod::p2", "qty": 1, "price": 49.99 }
  ],
  "total": 109.97
}
```

### Reference when:
- The nested entity is **accessed independently** (e.g., a product catalog entry)
- The nested entity is **updated frequently** and independently
- There's a **many-to-many** relationship
- The nested collection is **unbounded** (e.g., all posts by a user)

```json
{
  "type": "comment",
  "id": "cmt-001",
  "postId": "post::p42",   // reference to parent
  "userId": "user::u1",    // reference to user
  "body": "Great article!",
  "createdAt": "2024-03-15T12:00:00Z"
}
```

## Common Patterns

### User Profile
```json
{
  "type": "user",
  "email": "alice@example.com",
  "profile": { "firstName": "Alice", "lastName": "Chen", "avatarUrl": "..." },
  "preferences": { "theme": "dark", "notifications": true },
  "roles": ["admin", "editor"],
  "createdAt": "2024-01-01T00:00:00Z"
}
```

### Time-Series Event
```json
{
  "type": "event",
  "userId": "user::u1",
  "event": "page_view",
  "url": "/products/p1",
  "timestamp": "2024-03-15T14:32:01.123Z",
  "sessionId": "sess-xyz",
  "metadata": { "browser": "Chrome", "country": "US" }
}
```
Key: `event::2024-03-15::uuid` — sortable chronologically, expires via TTL after retention period.

### Product Catalog with Variants
```json
{
  "type": "product",
  "sku": "HDPH-001",
  "name": "Wireless Headphones",
  "category": "electronics",
  "basePrice": 149.99,
  "variants": [
    { "color": "black", "stock": 42, "sku": "HDPH-001-BLK" },
    { "color": "white", "stock": 18, "sku": "HDPH-001-WHT" }
  ],
  "description": "...",
  "tags": ["audio", "wireless", "sale"]
}
```

## Versioning

For schema evolution, embed a version field:

```json
{ "type": "user", "schemaVersion": 2, ... }
```

Then migrate lazily on read:
```python
doc = result.content_as[dict]
if doc.get("schemaVersion", 1) < 2:
    doc = migrate_v1_to_v2(doc)
```

For detailed patterns (hierarchical data, many-to-many, multi-tenancy, large document splitting), see `references/modeling-patterns.md`.
