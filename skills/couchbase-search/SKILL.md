---
name: couchbase-search
description: Use Couchbase Full-Text Search (FTS) and Vector Search. Use when performing text searches, fuzzy matching, geospatial queries, semantic/similarity search with embeddings, or hybrid search combining text and vector queries. Requires a search index to exist on the target scope.
allowed-tools: Bash, Read, Write, Edit
---

# Couchbase Search (FTS + Vector)

Couchbase Search runs on the Search Service and supports:
- **Full-Text Search (FTS)** — text analysis, fuzzy matching, wildcard, phrase, geospatial
- **Vector Search** — k-nearest-neighbor (kNN) over embedding vectors
- **Hybrid Search** — combine text and vector queries in a single request

Search indexes are defined at the **scope level** and search across one or more collections.

## Creating a Search Index

### Via Couchbase UI
Cluster → Search → Create Index → choose scope/collection → configure field mappings.

### Via REST API
```bash
curl -u user:pass -XPUT \
  http://localhost:8094/api/bucket/my-bucket/scope/my-scope/index/my-fts-index \
  -H "Content-Type: application/json" \
  -d @index-definition.json
```

### Minimal Index Definition (JSON)
```json
{
  "name": "my-fts-index",
  "type": "fulltext-index",
  "params": {
    "doc_config": {
      "mode": "scope.collection.type_field",
      "type_field": "type"
    },
    "mapping": {
      "default_mapping": { "enabled": true, "dynamic": true },
      "types": {
        "my-scope.products": {
          "enabled": true,
          "properties": {
            "description": { "fields": [{ "type": "text", "analyzer": "en", "index": true, "store": true }] },
            "name":        { "fields": [{ "type": "text", "analyzer": "en", "index": true, "store": true }] },
            "embedding":   { "fields": [{ "type": "vector", "dims": 1536, "similarity": "dot_product" }] }
          }
        }
      }
    }
  },
  "sourceType": "gocbcore",
  "sourceName": "my-bucket"
}
```

## Full-Text Search (Python SDK v4.x)

```python
from couchbase.search import SearchRequest, SearchOptions, MatchQuery, MatchPhraseQuery
from couchbase.search import FuzzyQuery, WildcardQuery, TermRangeQuery, NumericRangeQuery
import couchbase.search as search

# Simple match query (analyzed — handles stemming, case)
result = scope.search(
    "my-fts-index",
    SearchRequest.create(search.MatchQuery("wireless headphones")),
    SearchOptions(limit=10, fields=["name", "description", "price"])
)
for row in result.rows():
    print(row.id, row.score, row.fields)

# Phrase query — exact phrase match
result = scope.search(
    "my-fts-index",
    SearchRequest.create(search.MatchPhraseQuery("noise cancelling")),
    SearchOptions(limit=10)
)

# Fuzzy query — handles typos (fuzziness=1 or 2)
result = scope.search(
    "my-fts-index",
    SearchRequest.create(search.FuzzyQuery("headphons", fuzziness=1)),
    SearchOptions(limit=10)
)

# Compound query — boolean AND/OR/NOT
compound = search.ConjunctionQuery(
    search.MatchQuery("headphones"),
    search.NumericRangeQuery(min=50, max=300, field="price")
)
result = scope.search("my-fts-index", SearchRequest.create(compound), SearchOptions(limit=10))
```

## Vector Search (Semantic / Similarity)

Requires a vector field in the search index with `"type": "vector"` and the correct `dims`.

```python
from couchbase.vector_search import VectorQuery, VectorSearch

# Generate embedding for the query (use your embedding model)
query_embedding = embed_text("wireless noise-cancelling headphones")  # list[float]

result = scope.search(
    "my-fts-index",
    SearchRequest.create(
        VectorSearch.from_vector_query(
            VectorQuery("embedding", query_embedding, num_candidates=20)
        )
    ),
    SearchOptions(limit=10, fields=["name", "description"])
)
for row in result.rows():
    print(row.id, row.score, row.fields.get("name"))
```

## Hybrid Search (Text + Vector)

Combine FTS and vector in a single query — results are scored and merged:

```python
from couchbase.vector_search import VectorQuery, VectorSearch

query_text = "noise cancelling headphones"
query_vec  = embed_text(query_text)

result = scope.search(
    "my-fts-index",
    SearchRequest.create(
        search.MatchQuery(query_text)
    ).with_vector_search(
        VectorSearch.from_vector_query(
            VectorQuery("embedding", query_vec, num_candidates=20)
        )
    ),
    SearchOptions(limit=10, fields=["name", "description"])
)
```

## Geospatial Search

```python
# Documents must have a geo-point field: { "location": { "lat": 37.7, "lon": -122.4 } }
geo_query = search.GeoDistanceQuery(
    distance="10km",
    location=[37.7749, -122.4194],  # [lat, lon]
    field="location"
)
result = scope.search("my-fts-index", SearchRequest.create(geo_query), SearchOptions(limit=20))
```

## Search Options

```python
SearchOptions(
    limit=10,               # max results
    skip=0,                 # pagination offset
    fields=["name", "id"],  # fields to return in row.fields
    highlight=search.HighlightStyle.HTML,  # highlight matching terms
    score="none",           # omit scores for faster non-ranked retrieval
    timeout=timedelta(seconds=10),
)
```

For index configuration details, facets, sorting, and advanced query types, see `references/fts-guide.md`.
For vector index setup, embedding strategies, and hybrid ranking, see `references/vector-search-guide.md`.
