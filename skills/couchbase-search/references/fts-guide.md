# Full-Text Search (FTS) Guide

## Index Configuration

A search index must be created before running queries. Configure via the Couchbase UI (Cluster → Search → Create Index) or the REST API.

### Index Definition Structure

```json
{
  "name": "products-fts-index",
  "type": "fulltext-index",
  "params": {
    "doc_config": {
      "mode": "scope.collection.type_field",
      "type_field": "type",
      "docid_prefix_delim": "::",
      "docid_regexp": ""
    },
    "mapping": {
      "default_analyzer": "standard",
      "default_datetime_parser": "dateTimeOptional",
      "default_field": "_all",
      "default_mapping": {
        "enabled": false,
        "dynamic": false
      },
      "types": {
        "my-scope.products": {
          "enabled": true,
          "dynamic": false,
          "properties": {
            "name": {
              "fields": [{
                "type": "text",
                "analyzer": "en",
                "index": true,
                "store": true,
                "include_in_all": true,
                "include_term_vectors": true
              }]
            },
            "description": {
              "fields": [{
                "type": "text",
                "analyzer": "en",
                "index": true,
                "store": true
              }]
            },
            "price": {
              "fields": [{ "type": "number", "index": true }]
            },
            "createdAt": {
              "fields": [{ "type": "datetime", "index": true }]
            },
            "category": {
              "fields": [{ "type": "keyword", "index": true, "store": true }]
            },
            "location": {
              "fields": [{ "type": "geopoint", "index": true }]
            }
          }
        }
      }
    }
  },
  "sourceType": "gocbcore",
  "sourceName": "my-bucket"
}
```

**Field types:**
- `text` — analyzed text (use for search; choose analyzer like `en`, `standard`, `keyword`)
- `keyword` — exact match only (use for facets, filters on enums/IDs)
- `number` — numeric range queries
- `datetime` — date range queries (ISO 8601 format)
- `boolean` — true/false
- `geopoint` — lat/lon for geospatial queries
- `vector` — float32 array for vector similarity search

## Text Analyzers

| Analyzer | Use | Behavior |
|----------|-----|---------|
| `standard` | General text | Lowercase, remove punctuation, no stemming |
| `en` | English text | Lowercase + English stemming + stop words |
| `keyword` | Exact match | No transformation |
| `web` | Web content | HTML-aware |
| `simple` | Simple text | Lowercase only |

## Query Types (Python SDK)

```python
import couchbase.search as search
from couchbase.search import SearchRequest, SearchOptions

# MatchQuery — analyzed: handles stemming, case, stop words
search.MatchQuery("wireless headphones", field="description")
search.MatchQuery("headphones", analyzer="en", fuzziness=1)

# TermQuery — exact token match (no analysis)
search.TermQuery("electronics", field="category")

# MatchPhraseQuery — exact phrase (words in order)
search.MatchPhraseQuery("noise cancelling headphones", field="description")

# PrefixQuery — starts with (fast, but only on keyword fields)
search.PrefixQuery("electro", field="category")

# WildcardQuery — glob patterns (* = any, ? = one)
search.WildcardQuery("head*", field="name")

# RegexpQuery — regex match
search.RegexpQuery("[a-z]+\\d{3}", field="sku")

# TermRangeQuery — lexicographic range on keyword/string field
search.TermRangeQuery(start="b", end="m", field="name")

# NumericRangeQuery — numeric range
search.NumericRangeQuery(min=50, max=200, inclusive_min=True, inclusive_max=False, field="price")

# DateRangeQuery — date range
search.DateRangeQuery(
    start="2024-01-01T00:00:00Z",
    end="2024-12-31T23:59:59Z",
    field="createdAt"
)

# GeoDistanceQuery — within a radius
search.GeoDistanceQuery(distance="5km", location=[37.7749, -122.4194], field="location")
# Location is [lat, lon] or {"lat": 37.7, "lon": -122.4}

# GeoBoundingBoxQuery — within a bounding box
search.GeoBoundingBoxQuery(
    top_left=[37.8, -122.5],
    bottom_right=[37.7, -122.3],
    field="location"
)

# ConjunctionQuery — AND (all queries must match)
search.ConjunctionQuery(
    search.MatchQuery("headphones"),
    search.NumericRangeQuery(min=50, max=300, field="price")
)

# DisjunctionQuery — OR (at least min queries must match)
search.DisjunctionQuery(
    search.MatchQuery("headphones"),
    search.MatchQuery("earphones"),
    min=1
)

# BooleanQuery — combine MUST, SHOULD, MUST_NOT
bool_q = search.BooleanQuery()
bool_q.must = search.ConjunctionQuery(search.MatchQuery("headphones"))
bool_q.should = search.DisjunctionQuery(search.MatchQuery("wireless"), search.MatchQuery("bluetooth"))
bool_q.must_not = search.DisjunctionQuery(search.TermQuery("discontinued", field="status"))
```

## Facets

Facets aggregate search results for navigation (e.g., filter panels):

```python
from couchbase.search import SearchOptions, TermFacet, NumericFacet, DateFacet

result = scope.search(
    "products-fts-index",
    SearchRequest.create(search.MatchQuery("headphones")),
    SearchOptions(
        limit=10,
        facets={
            "by-category": TermFacet("category", limit=10),
            "by-price": NumericFacet("price", limit=5, ranges=[
                search.NumericRange("budget", 0, 50),
                search.NumericRange("mid-range", 50, 200),
                search.NumericRange("premium", 200, None),
            ]),
            "by-date": DateFacet("createdAt", limit=3, ranges=[
                search.DateRange("last-week", "2024-03-08T00:00:00Z", "2024-03-15T00:00:00Z"),
            ]),
        }
    )
)

for name, facet_result in result.facets().items():
    print(f"\n{name}:")
    for term in facet_result.terms:
        print(f"  {term.term}: {term.count}")
```

## Highlighting

```python
from couchbase.search import HighlightStyle

result = scope.search(
    "products-fts-index",
    SearchRequest.create(search.MatchQuery("wireless headphones")),
    SearchOptions(
        limit=10,
        fields=["name", "description"],
        highlight=HighlightStyle.HTML,   # wraps matches in <mark>
        # or HighlightStyle.ANSI for terminal output
    )
)
for row in result.rows():
    print(row.id, row.score)
    for field, fragments in row.fragments.items():
        print(f"  {field}: {fragments}")
```

## Sorting

```python
from couchbase.search import SearchSortScore, SearchSortField, SearchSortGeoDistance

result = scope.search(
    "products-fts-index",
    SearchRequest.create(search.MatchQuery("headphones")),
    SearchOptions(
        sort=[
            SearchSortScore(descending=True),       # primary: relevance score
            SearchSortField("price", descending=False),  # secondary: price asc
        ]
    )
)

# Sort by geo distance
result = scope.search(
    "products-fts-index",
    SearchRequest.create(search.GeoDistanceQuery("10km", [37.7749, -122.4194], field="location")),
    SearchOptions(
        sort=[SearchSortGeoDistance("location", [37.7749, -122.4194], unit="km")]
    )
)
```

## Index Management via REST API

```bash
# Create/update index
curl -u user:pass -XPUT \
  http://localhost:8094/api/bucket/{bucket}/scope/{scope}/index/{indexName} \
  -H "Content-Type: application/json" \
  -d @index.json

# List indexes
curl -u user:pass http://localhost:8094/api/index

# Delete index
curl -u user:pass -XDELETE \
  http://localhost:8094/api/bucket/{bucket}/scope/{scope}/index/{indexName}

# Index count (number of indexed documents)
curl -u user:pass \
  http://localhost:8094/api/bucket/{bucket}/scope/{scope}/index/{indexName}/count
```
