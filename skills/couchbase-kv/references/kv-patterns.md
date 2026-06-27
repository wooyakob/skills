# Couchbase KV Patterns

## Error Handling

```python
from couchbase.exceptions import (
    DocumentNotFoundException,
    DocumentExistsException,
    CasMismatchException,
    TimeoutException,
    CouchbaseException
)

def safe_get(collection, key):
    try:
        return collection.get(key).content_as[dict]
    except DocumentNotFoundException:
        return None
    except TimeoutException:
        raise  # Let caller decide on retry
    except CouchbaseException as e:
        log.error("KV error getting %s: %s", key, e)
        raise

def upsert_with_retry(collection, key, doc, max_retries=3):
    for attempt in range(max_retries):
        try:
            collection.upsert(key, doc)
            return
        except TimeoutException:
            if attempt == max_retries - 1:
                raise
            time.sleep(0.1 * (2 ** attempt))  # exponential backoff
```

## Advanced Sub-Document Operations

```python
import couchbase.subdocument as SD

# Increment a counter atomically
collection.mutate_in("stats::global", [
    SD.increment("pageViews", 1),
    SD.increment("uniqueVisitors", 1),
])

# Remove a field
collection.mutate_in("user::u1", [SD.remove("temporaryToken")])

# Array operations
collection.mutate_in("user::u1", [
    SD.array_prepend("recentItems", "item-001"),  # prepend to front
    SD.array_append("recentItems", "item-999"),   # append to back
    SD.array_addunique("roles", "editor"),         # add if not present
    SD.array_insert("recentItems[1]", "item-002"),# insert at position
])

# Insert a new field (fails if field already exists)
collection.mutate_in("product::p1", [
    SD.insert("newField", "value")
])

# Replace a specific field (fails if field doesn't exist)
collection.mutate_in("order::o1", [
    SD.replace("status", "shipped")
])

# Upsert at a deep path (creates intermediate objects)
collection.mutate_in("user::u1", [
    SD.upsert("preferences.notifications.email", True, create_parents=True)
])
```

## Get-and-Lock (Pessimistic Locking)

Use when you need exclusive access during a read-modify-write cycle:

```python
from couchbase.options import GetAndLockOptions
from datetime import timedelta

try:
    # Lock for up to 15 seconds
    result = collection.get_and_lock("inventory::prod-001", timedelta(seconds=15))
    doc = result.content_as[dict]
    cas = result.cas

    doc["stock"] -= 1

    # Unlock by replacing with the locked CAS
    from couchbase.options import ReplaceOptions
    collection.replace("inventory::prod-001", doc, ReplaceOptions(cas=cas))

except Exception:
    # On error, explicitly unlock (or let the 15s TTL expire)
    collection.unlock("inventory::prod-001", cas)
    raise
```

Prefer CAS (optimistic locking) over `get_and_lock` for most cases — it scales better under concurrency.

## Node.js KV Operations

```javascript
const { DocumentNotFoundException } = require('couchbase')

// GET
try {
  const result = await collection.get('doc-key')
  console.log(result.content)
} catch (e) {
  if (e instanceof DocumentNotFoundException) {
    console.log('Not found')
  } else throw e
}

// UPSERT with TTL
await collection.upsert('session-abc', { userId: 'u1' }, { expiry: 3600 })

// Sub-document
const { MutateInSpec, LookupInSpec } = require('couchbase')

await collection.mutateIn('user::u1', [
  MutateInSpec.increment('loginCount', 1),
  MutateInSpec.arrayAppend('tags', 'verified'),
])

const result = await collection.lookupIn('user::u1', [
  LookupInSpec.get('name'),
  LookupInSpec.get('email'),
])
const name  = result.content[0].value
const email = result.content[1].value

// Multi-get
const keys = ['user::1', 'user::2', 'user::3']
const results = await Promise.all(
  keys.map(k => collection.get(k).catch(e =>
    e instanceof DocumentNotFoundException ? null : Promise.reject(e)
  ))
)
```

## Bulk Load Pattern (Python)

```python
from concurrent.futures import ThreadPoolExecutor
import json

def load_documents(collection, docs: list[dict], key_fn, workers=16):
    """Bulk-insert documents using a thread pool."""
    def upsert_one(doc):
        key = key_fn(doc)
        collection.upsert(key, doc)
        return key

    with ThreadPoolExecutor(max_workers=workers) as ex:
        futures = [ex.submit(upsert_one, doc) for doc in docs]
        results = [f.result() for f in futures]
    return results

# Usage:
docs = json.load(open("data.json"))
keys = load_documents(
    collection=my_collection,
    docs=docs,
    key_fn=lambda d: f"product::{d['sku']}"
)
print(f"Loaded {len(keys)} documents")
```

## Document Expiry Patterns

```python
from datetime import datetime, timedelta, timezone
from uuid import uuid4

# Session store — auto-expire after 1 hour of inactivity
def refresh_session(collection, session_id):
    collection.touch(session_id, timedelta(hours=1))

# One-time tokens — expire after 15 minutes
def create_token(collection, token_id, user_id):
    collection.insert(
        f"token::{token_id}",
        {"userId": user_id, "createdAt": datetime.now(timezone.utc).isoformat()},
        InsertOptions(expiry=timedelta(minutes=15))
    )

# Audit log — keep for 90 days then auto-delete
def write_audit(collection, event):
    collection.upsert(
        f"audit::{datetime.now(timezone.utc).date()}::{uuid4()}",
        event,
        UpsertOptions(expiry=timedelta(days=90))
    )
```

## Java SDK KV (Reference)

```java
// GET
try {
    GetResult result = collection.get("doc-key");
    JsonObject doc = result.contentAsObject();
} catch (DocumentNotFoundException e) {
    // not found
}

// UPSERT with TTL
collection.upsert("session-abc",
    JsonObject.create().put("userId", "u1"),
    UpsertOptions.upsertOptions().expiry(Duration.ofHours(1))
);

// Sub-document
collection.mutateIn("user::u1", Arrays.asList(
    MutateInSpec.increment("loginCount", 1),
    MutateInSpec.arrayAppend("tags", "verified")
));
```
