---
name: couchbase-kv
description: Perform Couchbase Key-Value (KV) operations including get, upsert, insert, replace, remove, sub-document operations, batch operations, and TTL management. Use for direct document access by key — KV ops are the fastest path in Couchbase (sub-millisecond at P99 over LAN). Requires an established Collection reference from couchbase-connect.
allowed-tools: Bash, Read, Write, Edit
---

# Couchbase Key-Value Operations

KV operations access documents directly by their key, bypassing the query engine. Use KV for known-key access; use SQL++ (`couchbase-query`) for searches.

## Core Operations (Python SDK v4.x)

```python
from couchbase.exceptions import DocumentNotFoundException, DocumentExistsException
from couchbase.options import GetOptions, UpsertOptions, InsertOptions, ReplaceOptions
from datetime import timedelta

# GET — retrieve a document
try:
    result = collection.get("doc-key-123")
    doc = result.content_as[dict]
except DocumentNotFoundException:
    doc = None

# UPSERT — create or overwrite
collection.upsert("doc-key-123", {"type": "user", "name": "Alice", "score": 42})

# INSERT — create only (raises DocumentExistsException if key exists)
try:
    collection.insert("doc-key-123", {"type": "user", "name": "Alice"})
except DocumentExistsException:
    pass  # key already exists

# REPLACE — update only (raises DocumentNotFoundException if key absent)
collection.replace("doc-key-123", {"type": "user", "name": "Alice", "score": 99})

# REMOVE — delete a document
collection.remove("doc-key-123")
```

## TTL (Time-To-Live / Expiry)

```python
from datetime import timedelta

# Set expiry on upsert
collection.upsert(
    "session-abc",
    {"userId": "u1", "token": "xyz"},
    UpsertOptions(expiry=timedelta(hours=1))
)

# Extend expiry on a get (get_and_touch)
result = collection.get_and_touch("session-abc", timedelta(hours=1))

# Touch — extend TTL without reading full document
collection.touch("session-abc", timedelta(hours=1))

# Get remaining TTL
result = collection.get("session-abc", GetOptions(with_expiry=True))
expiry = result.expiry_time  # datetime
```

## Sub-Document Operations

Efficiently read or modify parts of a document without fetching/sending the whole thing.

```python
import couchbase.subdocument as SD

# lookup_in — read specific fields (faster than full get for large docs)
result = collection.lookup_in("doc-key-123", [
    SD.get("name"),
    SD.get("address.city"),
    SD.exists("premium"),
    SD.count("tags"),
])
name  = result.content_as[str](0)
city  = result.content_as[str](1)
has_premium = result.exists(2)
tag_count   = result.content_as[int](3)

# mutate_in — update specific fields atomically
collection.mutate_in("doc-key-123", [
    SD.upsert("status", "active"),
    SD.increment("loginCount", 1),
    SD.array_append("tags", "vip"),
    SD.array_addunique("roles", "editor"),
])
```

## Compare-And-Swap (CAS) — Optimistic Locking

```python
from couchbase.exceptions import CasMismatchException

# CAS prevents lost updates under concurrent writes
result = collection.get("doc-key-123")
doc = result.content_as[dict]
cas = result.cas

doc["score"] += 10
try:
    collection.replace("doc-key-123", doc, ReplaceOptions(cas=cas))
except CasMismatchException:
    # Another writer modified the doc; re-read and retry
    pass
```

## Batch Operations

```python
# get_multi — fetch multiple docs in a single round-trip
keys = ["user::1", "user::2", "user::3"]
results = collection.get_multi(keys)

for key, result in results.results.items():
    if result.success:
        print(key, result.value.content_as[dict])
    else:
        print(key, "not found or error:", result.exception)

# upsert_multi — write multiple docs
docs = {
    "user::1": {"name": "Alice"},
    "user::2": {"name": "Bob"},
}
collection.upsert_multi(docs)
```

## Key Design Conventions

Choose a key format and apply it consistently:

```
{type}::{id}              →  user::a1b2c3
{type}::{tenant}::{id}    →  order::acme::00042
{type}::{date}::{id}      →  log::2024-03-15::uuid
```

- Keep keys under 250 bytes
- Avoid special characters except `:`, `-`, `_`, `.`
- Include the document type in the key for easy debugging

## Durability

```python
from couchbase.durability import DurabilityLevel

# Majority — write acknowledged by majority of replicas (recommended for production)
collection.upsert(
    "doc-key",
    {"field": "value"},
    UpsertOptions(durability=DurabilityLevel.MAJORITY)
)
```

| Level | Guarantees |
|-------|-----------|
| `NONE` | In-memory acknowledgment (fastest) |
| `MAJORITY` | Majority of replicas in-memory (recommended) |
| `MAJORITY_AND_PERSIST_TO_ACTIVE` | Active node persisted to disk |
| `PERSIST_TO_MAJORITY` | Majority persisted to disk (safest, slowest) |

For Node.js equivalents, batch patterns, and error handling, see `references/kv-patterns.md`.
