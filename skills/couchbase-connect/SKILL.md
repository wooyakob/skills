---
name: couchbase-connect
description: Connect to a Couchbase cluster (self-managed or Capella cloud) using Python, Node.js, or Java SDKs. Use when setting up a Couchbase connection, configuring cluster options, obtaining bucket/scope/collection references, handling authentication, or troubleshooting connection issues.
allowed-tools: Bash, Read, Write, Edit
---

# Couchbase Connection

## Data Hierarchy

Couchbase organizes data as: **Cluster → Bucket → Scope → Collection → Document**

| Level | Analogy | Notes |
|-------|---------|-------|
| Cluster | Server / cloud endpoint | One connection per app |
| Bucket | Database | Top-level container |
| Scope | Schema | Namespace within a bucket |
| Collection | Table | Document group within a scope |
| Document | Row | JSON object with a unique string key |

The `_default` scope and `_default` collection exist in every bucket for backward compatibility.

## Connection Strings

| Environment | Scheme | Example |
|------------|--------|---------|
| Self-managed, no TLS | `couchbase://` | `couchbase://localhost` |
| Self-managed, TLS | `couchbases://` | `couchbases://db.example.com` |
| Capella (always TLS) | `couchbases://` | `couchbases://cb.<id>.cloud.couchbase.com` |

Multi-node: `couchbase://node1,node2,node3` — provide any one node; the SDK discovers the rest.

## Python SDK (v4.x)

```bash
pip install couchbase
```

```python
from couchbase.cluster import Cluster
from couchbase.auth import PasswordAuthenticator
from couchbase.options import ClusterOptions
from datetime import timedelta
import os

cluster = Cluster(
    os.environ["COUCHBASE_CONNECTION_STRING"],
    ClusterOptions(PasswordAuthenticator(
        os.environ["COUCHBASE_USERNAME"],
        os.environ["COUCHBASE_PASSWORD"]
    ))
)
cluster.wait_until_ready(timedelta(seconds=10))

bucket     = cluster.bucket("my-bucket")
scope      = bucket.scope("my-scope")           # or bucket.default_scope()
collection = scope.collection("my-collection")  # or bucket.default_collection()
```

## Node.js SDK (v4.x)

```bash
npm install couchbase
```

```javascript
const { connect } = require('couchbase')

const cluster = await connect(process.env.COUCHBASE_CONNECTION_STRING, {
  username: process.env.COUCHBASE_USERNAME,
  password: process.env.COUCHBASE_PASSWORD,
})

const bucket     = cluster.bucket('my-bucket')
const scope      = bucket.scope('my-scope')
const collection = scope.collection('my-collection')
```

## Java SDK (v3.x)

```xml
<dependency>
  <groupId>com.couchbase.client</groupId>
  <artifactId>java-client</artifactId>
  <version>3.7.3</version>
</dependency>
```

```java
import com.couchbase.client.java.*;

Cluster cluster = Cluster.connect(
    System.getenv("COUCHBASE_CONNECTION_STRING"),
    System.getenv("COUCHBASE_USERNAME"),
    System.getenv("COUCHBASE_PASSWORD")
);

Bucket bucket = cluster.bucket("my-bucket");
bucket.waitUntilReady(Duration.ofSeconds(10));
Collection collection = bucket.scope("my-scope").collection("my-collection");
```

## Capella Setup Checklist

1. **Cluster → Connect** — copy the connection string (format: `couchbases://cb.<id>.cloud.couchbase.com`)
2. **Security → Database Access** — create a database user with appropriate roles
3. **Security → Allowed IPs** — add your IP (or `0.0.0.0/0` for development only)
4. **Create bucket** — Clusters → your cluster → Buckets → Add Bucket

## Connection Best Practices

- **Singleton** — create `Cluster` once at app startup; it manages an internal connection pool
- **`wait_until_ready()`** — always call before the first operation (Python/Java); prevents early-connection errors
- **Environment variables** — never hardcode credentials
- **`couchbases://` for TLS** — mandatory for Capella; recommended for all production deployments
- **Graceful shutdown** — call `cluster.close()` / `cluster.disconnect()` on app exit
- **WAN profile** — apply `opts.apply_profile("wan_development")` when connecting to Capella or remote clusters

For advanced options (TLS certs, timeouts, async/await, certificate auth), see `references/connection-options.md`.
