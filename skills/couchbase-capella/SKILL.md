---
name: couchbase-capella
description: Work with Couchbase Capella, the fully-managed cloud database service. Use when setting up a Capella cluster, configuring access credentials, using the Capella management API, working with Capella Columnar (analytics), or setting up App Services for mobile/edge sync.
---

# Couchbase Capella

Capella is Couchbase's fully-managed Database-as-a-Service (DBaaS) on AWS, Azure, and GCP. It provides operational clusters (Couchbase Server), Columnar (analytical), and App Services (mobile sync) — all managed through the Capella portal or API.

## Capella Portal

https://cloud.couchbase.com

Key navigation:
- **Organizations** → select your org
- **Projects** → logical grouping of clusters
- **Clusters** → operational database clusters
- **Columnar** → analytical clusters (separate from operational)
- **App Services** → mobile/edge sync endpoint

## Operational Cluster: Quick Setup

1. **Create a cluster**: Projects → Create Cluster → choose cloud + region + instance type
2. **Create a bucket**: Cluster → Buckets → Add Bucket
3. **Create database credentials**: Cluster → Connect → Database Access → Create Credentials
4. **Allowlist your IP**: Cluster → Connect → Allowed IP Addresses → Add Allowed IP
5. **Get connection string**: Cluster → Connect → copy the `couchbases://cb.<id>.cloud.couchbase.com` string

## Connecting to Capella

```python
import os
from couchbase.cluster import Cluster
from couchbase.auth import PasswordAuthenticator
from couchbase.options import ClusterOptions
from datetime import timedelta

cluster = Cluster(
    os.environ["COUCHBASE_CONNECTION_STRING"],  # couchbases://cb.<id>.cloud.couchbase.com
    ClusterOptions(PasswordAuthenticator(
        os.environ["COUCHBASE_USERNAME"],
        os.environ["COUCHBASE_PASSWORD"]
    ))
)
cluster.wait_until_ready(timedelta(seconds=15))
```

**Required environment variables:**
```
COUCHBASE_CONNECTION_STRING=couchbases://cb.abc123def456.cloud.couchbase.com
COUCHBASE_USERNAME=db-user
COUCHBASE_PASSWORD=s3cur3p@ss
COUCHBASE_BUCKET=my-bucket
```

## Capella Management API

The REST API at `https://cloudapi.cloud.couchbase.com/v4/` manages clusters, buckets, users, and allowlists programmatically.

### Authentication

1. Generate an API key: Capella UI → Organization → API Keys → Create API Key
2. The key has an `id` and `secret`; use HTTP Basic Auth: `Authorization: Basic <base64(id:secret)>` (or pass `-u "id:secret"` to curl)

For V4 API, use an access token obtained via:
```bash
curl -u "api-key-id:api-key-secret" \
  https://cloudapi.cloud.couchbase.com/v4/organizations
```

### Common API Operations

```bash
BASE=https://cloudapi.cloud.couchbase.com/v4
AUTH="-u 'key-id:key-secret'"

# List organizations
curl $AUTH $BASE/organizations

# List clusters in a project
curl $AUTH $BASE/organizations/{orgId}/projects/{projId}/clusters

# Create an allowed IP
curl $AUTH -XPOST \
  $BASE/organizations/{orgId}/projects/{projId}/clusters/{clusterId}/allowedcidrs \
  -H "Content-Type: application/json" \
  -d '{"cidr": "203.0.113.0/24", "comment": "office"}'

# Create database credentials
curl $AUTH -XPOST \
  $BASE/organizations/{orgId}/projects/{projId}/clusters/{clusterId}/users \
  -H "Content-Type: application/json" \
  -d '{"name": "app-user", "password": "...", "access": [{"privileges": ["data_reader","data_writer"], "resources": {"buckets": [{"name": "my-bucket"}]}}]}'
```

## Capella Columnar (Analytics)

Capella Columnar is a separate analytical cluster optimized for complex SQL++ queries over large datasets. It can link to operational clusters to mirror data automatically.

```python
# Connection uses same SDK but points to Columnar endpoint
from couchbase.cluster import Cluster
from couchbase.auth import PasswordAuthenticator
from couchbase.options import ClusterOptions

cluster = Cluster(
    "couchbases://cb.<columnar-id>.cloud.couchbase.com",
    ClusterOptions(PasswordAuthenticator("user", "pass"))
)

# Analytics queries run via cluster.analytics_query()
result = cluster.analytics_query(
    "SELECT category, COUNT(*) AS cnt, AVG(price) AS avgPrice "
    "FROM `my-bucket`.`scope`.`products` GROUP BY category ORDER BY cnt DESC"
)
for row in result:
    print(row)
```

Columnar advantages: columnar storage, no impact on operational cluster, complex aggregations, long-running queries, direct S3/GCS data access.

## App Services (Mobile / Edge Sync)

App Services provides a sync endpoint for Couchbase Lite (mobile/edge SDK), enabling offline-first apps with automatic conflict resolution.

- Configured per-cluster in Capella UI: Cluster → App Services → New App Service
- Define **sync functions** (JavaScript) to control document routing and access
- Mobile clients use Couchbase Lite SDK (iOS/Android/Flutter/.NET)
- REST API available for server-to-App-Services communication

```javascript
// Example sync function (App Services)
function sync(doc, oldDoc) {
  if (doc.type === "message") {
    channel("channel." + doc.channelId);
    access(doc.userId, "channel." + doc.channelId);
  }
  requireAccess(doc.channelId ? "channel." + doc.channelId : "public");
}
```

For detailed Capella API reference, Terraform provider usage, and Columnar link setup, see `references/capella-guide.md`.
