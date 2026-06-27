# Couchbase Connection Options

## Python SDK: Advanced ClusterOptions

```python
from couchbase.cluster import Cluster
from couchbase.auth import PasswordAuthenticator
from couchbase.options import ClusterOptions, ClusterTimeoutOptions

auth = PasswordAuthenticator("username", "password")
opts = ClusterOptions(auth)

# Apply WAN profile — increases all timeouts for cloud/remote connections (recommended for Capella)
opts.apply_profile("wan_development")
# Equivalent to manually setting:
#   kv_timeout=20s, query_timeout=75s, views_timeout=75s,
#   search_timeout=75s, analytics_timeout=75s, connect_timeout=20s

cluster = Cluster("couchbases://cb.abc123.cloud.couchbase.com", opts)
cluster.wait_until_ready(timedelta(seconds=15))
```

## Custom Timeouts

```python
from couchbase.options import ClusterTimeoutOptions

timeout_opts = ClusterTimeoutOptions(
    kv_timeout=timedelta(seconds=5),
    query_timeout=timedelta(seconds=30),
    search_timeout=timedelta(seconds=30),
    connect_timeout=timedelta(seconds=15),
    management_timeout=timedelta(seconds=30),
)
opts = ClusterOptions(auth, timeout_options=timeout_opts)
```

## TLS: Custom Certificate (Self-Managed Clusters)

```python
from couchbase.options import ClusterOptions
from couchbase.auth import PasswordAuthenticator

# Point to CA certificate file (PEM format)
cluster = Cluster(
    "couchbases://my-server.example.com",
    ClusterOptions(
        PasswordAuthenticator("user", "pass"),
        **{"trust_store_path": "/path/to/ca.pem"}
    )
)
```

For Capella, the Digicert root CA is trusted by default — no custom cert needed.

## Certificate Authentication (mTLS)

```python
from couchbase.auth import CertificateAuthenticator

auth = CertificateAuthenticator(
    cert_path="/path/to/client-cert.pem",
    key_path="/path/to/client-key.pem"
)
cluster = Cluster("couchbases://...", ClusterOptions(auth))
```

## Async Python (asyncio / FastAPI / aiohttp)

Use `acouchbase` — the async variant of the SDK (same package, different import):

```python
from acouchbase.cluster import AsyncCluster
from couchbase.auth import PasswordAuthenticator
from couchbase.options import ClusterOptions
from datetime import timedelta

async def create_cluster():
    cluster = AsyncCluster(
        "couchbases://cb.abc123.cloud.couchbase.com",
        ClusterOptions(PasswordAuthenticator("user", "pass"))
    )
    await cluster.wait_until_ready(timedelta(seconds=10))
    return cluster

# All operations on AsyncCluster are awaitables
async def get_doc(collection, key):
    result = await collection.get(key)
    return result.content_as[dict]

# Cleanup
async def shutdown(cluster):
    await cluster.close()
```

**FastAPI example:**
```python
from fastapi import FastAPI
from acouchbase.cluster import AsyncCluster
from couchbase.auth import PasswordAuthenticator
from couchbase.options import ClusterOptions
from contextlib import asynccontextmanager

_cluster = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global _cluster
    _cluster = AsyncCluster("couchbases://...", ClusterOptions(PasswordAuthenticator("u", "p")))
    await _cluster.wait_until_ready(timedelta(seconds=10))
    yield
    await _cluster.close()

app = FastAPI(lifespan=lifespan)
```

## Node.js: Advanced Options

```javascript
const cluster = await connect('couchbases://cb.abc123.cloud.couchbase.com', {
  username: process.env.COUCHBASE_USERNAME,
  password: process.env.COUCHBASE_PASSWORD,
  timeouts: {
    kvTimeout:          2500,   // ms — KV op timeout
    queryTimeout:       75000,  // ms — SQL++ query timeout
    searchTimeout:      30000,  // ms — FTS/vector timeout
    connectTimeout:     10000,  // ms — initial connection timeout
    managementTimeout:  30000,  // ms — admin API timeout
  },
  // TLS for self-managed with custom cert:
  // trustStorePath: '/path/to/ca.pem',
})
```

## Standard Environment Variables

Use these names for cross-service compatibility:

```bash
COUCHBASE_CONNECTION_STRING=couchbases://cb.abc123.cloud.couchbase.com
COUCHBASE_USERNAME=app-user
COUCHBASE_PASSWORD=s3cur3p@ssword
COUCHBASE_BUCKET=my-bucket
COUCHBASE_SCOPE=my-scope
COUCHBASE_COLLECTION=my-collection
```

## Singleton Pattern (Python)

```python
# db.py — import this module wherever you need Couchbase access
import os
from couchbase.cluster import Cluster
from couchbase.auth import PasswordAuthenticator
from couchbase.options import ClusterOptions
from datetime import timedelta

_cluster = None
_bucket  = None

def get_collection(scope: str = "_default", collection: str = "_default"):
    global _cluster, _bucket
    if _cluster is None:
        opts = ClusterOptions(PasswordAuthenticator(
            os.environ["COUCHBASE_USERNAME"],
            os.environ["COUCHBASE_PASSWORD"]
        ))
        opts.apply_profile("wan_development")
        _cluster = Cluster(os.environ["COUCHBASE_CONNECTION_STRING"], opts)
        _cluster.wait_until_ready(timedelta(seconds=15))
        _bucket = _cluster.bucket(os.environ["COUCHBASE_BUCKET"])
    return _bucket.scope(scope).collection(collection)
```

## Troubleshooting Connections

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `UnambiguousTimeoutException` on first op | `wait_until_ready` not called | Call before first operation |
| `AuthenticationException` | Wrong username/password | Verify Database Access credentials (not UI login) |
| `CouchbaseException: connection refused` | Wrong port or IP | Check connection string and Allowed IPs |
| TLS cert error | Self-signed cert not trusted | Add `trust_store_path` option |
| Slow queries only | Using `couchbase://` to Capella | Use `couchbases://` and WAN profile |
