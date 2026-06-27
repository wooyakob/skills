# Couchbase Capella Reference Guide

## Capella REST API (v4)

Base URL: `https://cloudapi.cloud.couchbase.com/v4`

### Authentication

Generate an API key pair in: Organization → API Keys → Create API Key

```python
import requests
from requests.auth import HTTPBasicAuth

API_KEY_ID     = "your-api-key-id"
API_KEY_SECRET = "your-api-key-secret"
BASE_URL       = "https://cloudapi.cloud.couchbase.com/v4"

auth = HTTPBasicAuth(API_KEY_ID, API_KEY_SECRET)

def api_get(path):
    r = requests.get(f"{BASE_URL}{path}", auth=auth)
    r.raise_for_status()
    return r.json()

def api_post(path, payload):
    r = requests.post(f"{BASE_URL}{path}", json=payload, auth=auth)
    r.raise_for_status()
    return r.json()
```

### Common API Operations

```python
# Get organization ID
orgs = api_get("/organizations")
org_id = orgs["data"][0]["id"]

# List projects
projects = api_get(f"/organizations/{org_id}/projects")

# List clusters in a project
proj_id = "your-project-id"
clusters = api_get(f"/organizations/{org_id}/projects/{proj_id}/clusters")

# Create a bucket
cluster_id = "your-cluster-id"
api_post(
    f"/organizations/{org_id}/projects/{proj_id}/clusters/{cluster_id}/buckets",
    {
        "name": "my-bucket",
        "type": "couchbase",
        "storageBackend": "couchstore",
        "memoryAllocationInMb": 512,
        "bucketConflictResolution": "seqno",
        "durabilityLevel": "none",
        "replicas": 1,
        "flush": False,
        "timeToLiveInSeconds": 0
    }
)

# Create a scope
api_post(
    f"/organizations/{org_id}/projects/{proj_id}/clusters/{cluster_id}/buckets/my-bucket/scopes",
    {"name": "my-scope"}
)

# Create a collection
api_post(
    f"/organizations/{org_id}/projects/{proj_id}/clusters/{cluster_id}/buckets/my-bucket/scopes/my-scope/collections",
    {"name": "my-collection", "maxTTL": 0}
)

# Add allowed IP
api_post(
    f"/organizations/{org_id}/projects/{proj_id}/clusters/{cluster_id}/allowedcidrs",
    {"cidr": "0.0.0.0/0", "comment": "Allow all (dev only)"}
)

# Create database user
api_post(
    f"/organizations/{org_id}/projects/{proj_id}/clusters/{cluster_id}/users",
    {
        "name": "app-user",
        "password": "SecurePass123!",
        "access": [{
            "privileges": ["data_reader", "data_writer"],
            "resources": {
                "buckets": [{
                    "name": "my-bucket",
                    "scopes": [{"name": "*", "collections": ["*"]}]
                }]
            }
        }]
    }
)
```

## Terraform Provider

The official Couchbase Capella Terraform provider manages clusters, buckets, scopes, collections, and users as infrastructure-as-code.

```hcl
terraform {
  required_providers {
    couchbase-capella = {
      source  = "couchbasecloud/couchbase-capella"
      version = "~> 1.0"
    }
  }
}

provider "couchbase-capella" {
  authentication_token = var.capella_api_key
}

resource "couchbase-capella_cluster" "example" {
  organization_id = var.org_id
  project_id      = var.project_id
  name            = "my-cluster"
  cloud_provider = {
    type   = "aws"
    region = "us-east-1"
    cidr   = "10.0.0.0/23"
  }
  service_groups = [{
    node = {
      compute    = { cpu = 4, ram = 16 }
      disk       = { type = "gp3", storage = 64, iops = 3000 }
    }
    num_of_nodes = 3
    services     = ["data", "query", "index", "search"]
  }]
  availability    = { type = "multi" }
  support         = { plan = "developer pro", timezone = "PT" }
}

resource "couchbase-capella_bucket" "example" {
  organization_id = var.org_id
  project_id      = var.project_id
  cluster_id      = couchbase-capella_cluster.example.id
  name            = "my-bucket"
  type            = "couchbase"
  memory_allocation_in_mb = 512
  replicas        = 1
}
```

## Capella Columnar

Columnar is a separate analytical cluster for complex SQL++ analytics, powered by a columnar storage engine. It supports:
- **Live links** to operational Couchbase clusters (streaming replication)
- **External links** to S3, Azure Blob, GCS
- Complex aggregations, window functions, long-running queries
- No index required — full scans are optimized

### Columnar Connection

```python
from couchbase.cluster import Cluster
from couchbase.auth import PasswordAuthenticator
from couchbase.options import ClusterOptions

# Columnar has its own endpoint, separate from operational cluster
cluster = Cluster(
    "couchbases://cb.<columnar-id>.cloud.couchbase.com",
    ClusterOptions(PasswordAuthenticator("columnar-user", "pass"))
)

# Use analytics_query() for Columnar
result = cluster.analytics_query(
    """
    SELECT category,
           COUNT(*) AS productCount,
           AVG(price) AS avgPrice,
           PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY price) AS medianPrice
    FROM `operational-link`.`my-bucket`.`scope`.`products`
    GROUP BY category
    ORDER BY productCount DESC
    LIMIT 20
    """
)
for row in result:
    print(row)
```

### Setting Up a Columnar Link

In Columnar UI: Databases → your Columnar cluster → Links → Create Link → select your operational cluster.

Once linked, datasets are accessed as:
```sql
`link-name`.`bucket-name`.`scope-name`.`collection-name`
```

### Columnar SQL++ Extensions

```sql
-- CREATE ANALYTICS COLLECTION (shadow collection for schema control)
CREATE ANALYTICS COLLECTION my_products ON `link`.`bucket`.`scope`.`products`;

-- Window functions (not available in operational SQL++)
SELECT name, price,
  RANK() OVER (PARTITION BY category ORDER BY price DESC) AS rank_in_category,
  LAG(price) OVER (PARTITION BY category ORDER BY price) AS prev_price
FROM my_products;

-- External data (S3)
CREATE LINK s3_link TYPE S3 WITH {
  "accessKeyId": "...", "secretAccessKey": "...", "region": "us-east-1"
};
CREATE EXTERNAL DATASET s3_data ON `s3_link` AT "my-bucket/data/";
SELECT * FROM s3_data LIMIT 10;
```

## App Services (Mobile / Edge Sync)

App Services provides a sync gateway for Couchbase Lite (iOS, Android, Flutter, .NET).

### Creating an App Service

Capella UI → Cluster → App Services → New App Service → link to bucket

### Sync Function

```javascript
// Controls how documents are routed to sync channels
// and which users have access to which channels
function sync(doc, oldDoc) {
    // Route messages to per-conversation channels
    if (doc.type === "message") {
        var channel = "conversation." + doc.conversationId;
        
        // Grant all participants access
        if (doc.participants && Array.isArray(doc.participants)) {
            doc.participants.forEach(function(userId) {
                access(userId, channel);
            });
        }
        
        channel(channel);
        return;
    }

    // All other docs go to the user's private channel
    requireUser(doc.userId);
    channel("user." + doc.userId);
}
```

### App Services REST API

```bash
APP_SVC_URL=https://epd.<cluster-id>.apps.cloud.couchbase.com

# Get a document
curl -u "appuser:pass" $APP_SVC_URL/my-bucket/doc-key

# Put a document
curl -u "appuser:pass" -XPUT \
  $APP_SVC_URL/my-bucket/doc-key \
  -H "Content-Type: application/json" \
  -d '{"type": "message", "body": "Hello"}'

# Changes feed (real-time updates)
curl -u "appuser:pass" "$APP_SVC_URL/my-bucket/_changes?feed=longpoll&since=0"
```

## Capella Free Tier

Free Tier clusters are available for development:
- 1 node, shared infrastructure
- Connection string: `couchbases://cb.<id>.cloud.couchbase.com`
- 1 GB RAM, 2.5 GB storage
- Same SDK connection process as paid clusters

Create at: https://cloud.couchbase.com → Try Free
